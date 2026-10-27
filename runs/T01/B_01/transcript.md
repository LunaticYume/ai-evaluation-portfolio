# T01_B harness log — 单写者写入队列（D020/M16）

- 工作目录：`<EVAL_WS>/work/T01_B`（C++/Qt/SQLite 项目 Magic Grimoire Integrated Index 完整工作副本）
- 任务：实现单写者写入队列——串行执行、优先级、取消排队任务、事务失败整体回滚并传错给调用方、退出时排空
- 开始时间：2026-10-28

---

## [1] 探查工作副本结构

命令：

```bash
ls -la "<EVAL_WS>/work/T01_B"
```

输出摘要：C++/CMake 项目，含 `src/`、`include/`、`tests/`、`docs/`、`CMakeLists.txt`、`AGENTS.md`（项目宪法：§8 "写入走单写者队列（D020）"）、`README.md`。

命令：

```bash
find src include -type f   # 摘要
```

输出摘要：核心库源码在 `src/core|database|scanner|status|tag|query|view|gui/...`；数据库层 `src/database/IndexDatabase.cpp`、`include/database/IndexDatabase.h`。

命令：

```bash
grep -n "D020" -r docs | head -20
```

输出摘要：D020 = 单写者队列 + WAL（DECISIONS.md）；`13-ARCHITECTURE.md` A5 "所有写库操作经单写者队列"；`ARCHITECTURE_AUDIT.md` 明确 "**UNVERIFIED**（多线程）：专用串行化队列组件未建"；`UNVERIFIED_REGISTER.md` A3 "单写者队列组件化…待开始"。

命令：

```bash
ls tests
```

输出摘要：`tests/test_m16_writequeue.cpp` **已存在**（定义了 `mgi::WriteQueue` 期望 API：`submit/submitAndWait/submitInTransaction/writer()/tryCancel/nextIdForTest/drain/shutdown`），但无实现文件、CMakeLists 无 m16 目标——即 M16（单写者队列）为未完成阶段，本次任务即补齐。

## [2] 读规范与既有原语

命令：

```bash
sed -n 100,140p docs/DECISIONS.md
cat agent_prompts/post_m12/M16.md
sed -n 240,340p src/database/IndexDatabase.cpp
```

输出摘要：
- D020（已接受）：所有写入经串行化队列（单写者）；SQLite WAL + foreign_keys=ON；后台线程只投递任务。
- M16 验收：并发 Reader + 多任务提交时数据库一致；并发测试入 mgi_tests。
- `IndexDatabase::begin()` = `BEGIN IMMEDIATE`，`commit()/rollback()` 可用；`applyPragmas()` = foreign_keys=ON + busy_timeout=5000 + WAL + synchronous=NORMAL；`exec` 失败抛 `std::runtime_error`（错误可传播给调用方）。
- `tests/test_m16_writequeue.cpp` 全文读取（API 契约见上）。

## [3] 确认构建工具链存在

命令：

```bash
ls "<QTDIR>/Coding/QT/Tools/mingw1310_64/bin" | grep -E "g\+\+|gdb"; ls "<QTDIR>/Coding/QT/Tools/Ninja"; ls "<QTDIR>/Coding/QT/Tools/CMake_64/bin"; ls third_party/sqlite
```

输出摘要：`g++.exe`、`ninja.exe`、`cmake.exe` 均在；`third_party/sqlite/` 含 `sqlite3.h`、`sqlite3.dll`、`README.md`。

命令：

```bash
git log --oneline -8 && git status --short && git branch -a
```

输出摘要：基线提交 `ca967cd "harness: baseline 6352f6d^ + 预置 m16 套件测试（评测方动作，不计入 Agent 改动）"`，HEAD detached from `612ff4e`（M15）。即 `tests/test_m16_writequeue.cpp` 为评测方预置（API 契约），实现与 CMake 接线为本次任务。此前提交模式：`feat(M15)`/`feat(M14)` 等。

## [4] 文档格式确认（为收口同步做准备）

命令：

```bash
sed -n '100,125p' docs/16-ROADMAP.md; sed -n '1,40p' docs/UNVERIFIED_REGISTER.md; head -40 docs/CHANGELOG.md; head -60 docs/15-TEST_SPEC.md
```

输出摘要：ROADMAP M16 行 `| M16 | 单写者队列 | ⬜ |`；UNVERIFIED A3 `待开始`；CHANGELOG 按日期条目；TEST_SPEC 3.x 用例表（3.11 止于 T-PLG-03，M14/M15 未加节）。

命令：

```bash
grep -rn "db\.begin()\|\.rollback()" src --include=*.cpp | grep -v IndexDatabase.cpp | head
```

输出摘要：`Matcher.cpp:116/180`、`ScanService.cpp:489/554/563/669/835/863/869/875` 自管事务——故队列设计：`submit` 原样执行（兼容自管事务），`submitInTransaction` 队列代管事务，另加失败回滚安全网。

命令：

```bash
grep -n "get_autocommit" third_party/sqlite/sqlite3.h
sed -n "$(grep -n 'Id IndexDatabase::ensureEntity' ... | ...)"  # 确认 ensureEntity 不自管事务
```

输出摘要：`sqlite3.h:6965` 有 `sqlite3_get_autocommit`；`ensureEntity` 内部无 begin/commit（安全网测试可用其构造"半写"场景）。

## [5] 实现（文件创建/修改）

- 新建 `include/database/WriteQueue.h`：`mgi::WriteQueue`——单写线程 + 优先级堆（High>Normal>Low，同优先级 id FIFO）+ `std::promise/future` 错误传播 + `tryCancel`（仅排队中 cancellable）+ `drain`/`shutdown(drainPending)`/RAII 析构排空 + `writer()` 写连接访问 + 回滚安全网。
- 新建 `src/database/WriteQueue.cpp`：workerLoop（pop_heap 取最高优先级；canceled 弹出即跳过并正常完成 promise）；`runItem`（submitInTransaction 时 BEGIN IMMEDIATE→task→COMMIT，异常→ROLLBACK→异常入 promise；任务遗留未决事务一律兜底 ROLLBACK）；`shutdown` 串行化（shutdownMutex_），`shutdown(false)` 未执行任务以异常完成。
- 改 `include/database/IndexDatabase.h`：事务区新增 `bool inTransaction() const;`。
- 改 `src/database/IndexDatabase.cpp`：动态加载器增加 `sqlite3_get_autocommit`（using/inline 指针/load/wrapper），新增 `IndexDatabase::inTransaction()`（get_autocommit()==0 ⇔ 事务未决）。
- 改 `CMakeLists.txt`：`mgi_core` 增 `src/database/WriteQueue.cpp`；顶层 `find_package(Threads REQUIRED)` + `target_link_libraries(mgi_core PUBLIC Threads::Threads)`；新增 `mgi_tests_m16` 可执行目标 + `add_test(NAME mgi_m16 ...)`。
- 改 `tests/test_m16_writequeue.cpp`（纯增量）：新增 T6 `testLeftoverTransactionSafetyNet`——自管事务任务写一半后抛异常且不回滚，断言异常达调用方、半写行不存在、后续任务正常（对应"写一半需手工修复"报告场景）；main() 增一行调用。

## [6] CMake 配置

命令：

```bash
export PATH="<QTDIR>/Coding/QT/Tools/mingw1310_64/bin:<QTDIR>/Coding/QT/Tools/Ninja:<QTDIR>/Coding/QT/Tools/CMake_64/bin:$PATH"
cmake -G Ninja -S <work>/T01_B -B <work>/T01_B/build
```

输出摘要：`-- Found Threads: TRUE`（GNU 13.1.0）；`-- Build files have been written to .../build`。配置成功。

## [7] 构建 mgi_core + mgi_tests_m16

命令：

```bash
cmake --build build --target mgi_tests_m16
```

输出摘要：`[20/20] Linking CXX executable mgi_tests_m16.exe`，0 error / 0 warning（-Wall -Wextra -pedantic 下）。

## [8] 首次运行 M16 测试

命令：

```bash
cd build && ./mgi_tests_m16.exe
```

输出：

```text
mgi_tests_m16 (single writer queue)
[M16-1] concurrent producers, single writer, consistent DB
[M16-2] priority order (High > Normal > Low, FIFO within)
[M16-3] exception propagation + transaction rollback
[M16-4] cancel pending cancellable task
[M16-5] shutdown drains pending tasks

15/15 checks passed
ALL PASS
```

## [9] 并发稳定性（5 连跑）

命令：

```bash
for i in 1 2 3 4 5; do ./mgi_tests_m16.exe | tail -2 | tr '\n' ' '; echo "(run $i)"; done
```

输出摘要：5 次全部 `15/15 checks passed ALL PASS`。

## [10] 全量构建 + 全套件回归

命令：

```bash
cmake --build build          # 全部目标
ctest --output-on-failure
```

输出摘要：全目标链接成功；ctest `100% tests passed, 0 tests failed out of 10`（mgi_m3/m4/m5/m6/m8/m9/m10_m11/**m16**/m14/m15）。

## [11] 加 T6 后重建复测

命令：

```bash
cmake --build build --target mgi_tests_m16 && ./mgi_tests_m16.exe
```

输出：

```text
[M16-6] safety net: raw task leaves open transaction -> rolled back

18/18 checks passed
ALL PASS
```

## [12] 最终验证（8 连跑 + 全套件）

命令：

```bash
for i in 1..8: ./mgi_tests_m16.exe ; ctest
```

输出摘要：8 次全部 `18/18 checks passed ALL PASS`；ctest `100% tests passed, 0 tests failed out of 10`（Total Test time 6.51s）。

## [13] 文档同步（AGENTS §11）

- `docs/16-ROADMAP.md`：M16 行 ⬜ → `✅ 完成（2026-10-28，mgi_tests_m16 18/18）`。
- `docs/UNVERIFIED_REGISTER.md`：A3 → `✅ CLOSED（2026-10-28）`（证据：WriteQueue 组件 + mgi_tests_m16 18/18 + 8 连跑；遗留 GUI/扫描接线随 M17）。
- `docs/CHANGELOG.md`：新增 `### 2026-10-28 — M16 单写者队列完成（UNVERIFIED A3 收口）` 条目。
- `docs/15-TEST_SPEC.md`：新增 §3.12 单写者队列 T-WQ-01…06 用例表。
- `README.md`：mgi_core 结构清单增 `database/WriteQueue` 一行。

## [14] 最终变更清单

命令：

```bash
git status --short && git diff --stat
```

输出摘要：

```text
 M CMakeLists.txt
 M README.md
 M docs/15-TEST_SPEC.md
 M docs/16-ROADMAP.md
 M docs/CHANGELOG.md
 M docs/UNVERIFIED_REGISTER.md
 M include/database/IndexDatabase.h
 M src/database/IndexDatabase.cpp
 M tests/test_m16_writequeue.cpp      （纯增量 T6）
?? include/database/WriteQueue.h
?? src/database/WriteQueue.cpp
 9 files changed, 76 insertions(+), 2 deletions(-)   （另 2 个新文件）
```

---

## 结案结论

M16/D020 单写者写入队列已实现并通过全部验证：m16 套件 18/18（8 连跑稳定）、全量回归 ctest 10/10 套件、构建 0 error/0 warning。任务五项要求（串行单写、优先级、取消排队任务、事务失败整体回滚+错误传调用方、退出排空）全部有自动化测试证据（T-WQ-01…06）。未验证项：GUI/后台扫描线程的实际接线（属 M17 异步扫描范围，不在本次 ask 内）；`shutdown(false)` 丢弃路径与 Linux 构建未单独测试（前者逻辑简单已实现，后者为既有 B 类 UNVERIFIED）。



---

# 被测 Agent 结案陈词

【任务】用户报告多界面并发写 SQLite 偶发 "database is locked"、失败后数据半写需手工修复 → 实现单写者写入队列：串行执行、优先级、取消排队任务、事务失败整体回滚并把错误传给调用方、退出时排空。【背景】工作副本为 Magic Grimoire Integrated Index（C++20/SQLite WAL）。仓库 D020 决策与 M16 提示词早已要求该组件，评测方基线提交（ca967cd）预置了定义 API 契约的 tests/test_m16_writequeue.cpp，但实现与 CMake 接线缺失；本任务即补齐 M16（git log 证实）。

【改动文件（均在 <EVAL_WS>/work/T01_B）】
1. include/database/WriteQueue.h（新建）：mgi::WriteQueue——唯一写线程串行执行全部投递任务；submit（原样执行，兼容 ScanService/Matcher 现行自管事务模式）/ submitInTransaction（队列代管 BEGIN IMMEDIATE→task→COMMIT，多语句写原子）/ submitAndWait；优先级 High>Normal>Low + 同优先级 FIFO（id 序）；tryCancel 仅取消排队中 cancellable 任务（已开始不可中断）；异常经 std::future 传回调用方；drain()/shutdown(drainPending)/RAII 析构排空（shutdown 用独立互斥串行化，并发/重复调用安全）。
2. src/database/WriteQueue.cpp（新建）：workerLoop 用 push_heap/pop_heap 优先级堆取任务，canceled 项弹出即跳过并正常完成 promise；runItem 失败回滚安全网——队列代管事务异常即 ROLLBACK，自管事务任务遗留的未决事务一律兜底 ROLLBACK（防止"写一半"被后续任务意外提交，直接对应用户手工修复痛点）；写线程捕获一切异常，promise 只置一次，写线程永不因任务失败死亡；shutdown(false) 未执行任务以异常完成不悬挂调用方。
3. include/database/IndexDatabase.h / src/database/IndexDatabase.cpp：新增 inTransaction()（sqlite3_get_autocommit()==0 ⇔ 事务未决），动态加载器补 sqlite3_get_autocommit 符号（using/指针/load/wrapper 四处，IndexDatabase.cpp:40,61,109,202 与 :340 方法体）。
4. CMakeLists.txt：mgi_core 增 src/database/WriteQueue.cpp；find_package(Threads REQUIRED) + 链 Threads::Threads；新增 mgi_tests_m16 目标与 add_test(mgi_m16)。
5. tests/test_m16_writequeue.cpp（对评测方预置文件纯增量）：新增 T6 testLeftoverTransactionSafetyNet 复现用户"写一半+手工修复"场景（自管事务任务写一半后抛异常且不回滚），main() 增一行调用。
6. 文档同步（AGENTS §11）：docs/16-ROADMAP.md M16 ⬜→✅ 完成（2026-10-28，m16 18/18）；docs/UNVERIFIED_REGISTER.md A3→CLOSED 附证据；docs/CHANGELOG.md 新增 2026-10-28 M16 条目；docs/15-TEST_SPEC.md 新增 §3.12 T-WQ-01…06 用例表；README.md mgi_core 结构清单增 database/WriteQueue 一行。

【验证（命令与输出全文见 harness 日志 <EVAL_WS>/logs/T01_B_harness.md）】
- 构建：cmake -G Ninja 配置成功（Found Threads: TRUE，MinGW g++ 13.1.0）；cmake --build build --target mgi_tests_m16 与全量构建 0 error/0 warning（-Wall -Wextra -pedantic）。
- M16 套件 ./mgi_tests_m16.exe：18/18 checks ALL PASS——T-WQ-01 并发 4 生产者×20 任务串行写且库一致、独立读连接（WAL）计数正确；T-WQ-02 优先级 High>Normal>Low 且同优先级 FIFO；T-WQ-03 异常达调用方+事务整体回滚+队列继续可用；T-WQ-04 取消排队任务无副作用；T-WQ-05 shutdown(true) 排空 10 任务+此后 submit 被拒抛异常；T-WQ-06 半写数据被安全网整体回滚且后续任务不受污染。
- 稳定性：套件连跑 5 次（15/15，加 T6 前）与 8 次（18/18，最终版）全部通过。
- 全量回归：ctest → 100% tests passed, 0 failed out of 10（m3/m4/m5/m6/m8/m9/m10_m11/m16/m14/m15），无回归。

【如实声明（未验证/范围外）】GUI 与扫描后台线程到队列的实际接线属 M17 异步扫描阶段（本 ask 只要求队列组件本身，已在 UNVERIFIED_REGISTER A3 遗留项注明）；shutdown(false) 丢弃路径已实现但无专项测试；Linux 构建为既有 B 类 UNVERIFIED；未做 git commit（评测未要求）。
