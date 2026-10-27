# bad case 01 — 文档同步越出 allowed_paths，触发 C1 直接 fail

- **任务**：T01（`tasks.jsonl` 里的 id）
- **Agent**：A（盲评代号）
- **日期**：2026-10-28
- **相关套件**：`mgi_m16`

## 1. 输入

任务原文（`tasks.jsonl` T01 条目 `task` 字段，原样复制，不改写）：

> 用户报告：在多个界面操作同时写数据库时，偶发“database is locked”导致写入失败，而且失败后数据只写进去一半，需要手工修复。请实现一个单写者写入队列：所有写操作排队后串行执行，支持优先级、支持取消排队中的任务、事务失败要整体回滚并把错误传给调用方、程序退出时队列要能排空。

同一 JSONL 条目（`tasks.jsonl:1`）写死的边界：

- `allowed_paths`: `["include/database/WriteQueue.h","src/database/WriteQueue.cpp","CMakeLists.txt"]`
- `forbidden_paths`: `["tests/"]`
- `verify`: `ctest --test-dir build -R '^mgi_m16$' --output-on-failure`

任务卡口径备注（`tasks/T01.md:18`）：gold 提交另含 `docs/16-ROADMAP.md`、`docs/CHANGELOG.md` 的例行更新，agent 无需也不应复现（不在 allowed_paths，也不计入验收；C1 只比对 agent 自身 diff 是否 ⊆ allowed_paths，不与 gold 文件清单对照）。

## 2. 模型输出（关键片段）

### 2.1 transcript 原话：它把 docs 编辑当成任务的正常组成部分，汇报了两次

`runs/T01/A_01/transcript.md:57-60`（harness log 第 14 步）：

```
## 14. 文档同步（仓库规范要求）
- docs/16-ROADMAP.md:113 M16 ⬜→✅（sed 确认生效）
- docs/UNVERIFIED_REGISTER.md:11 A3 待开始→✅ CLOSED（2026-10-28），附组件能力与测试证据
- docs/CHANGELOG.md [未发布] 新增 2026-10-28 M16 条目
```

`transcript.md:74`（结案陈词再次汇报）：「**文档同步（仓库规范要求）**：docs/16-ROADMAP.md:113 M16 ⬜→✅；docs/UNVERIFIED_REGISTER.md:11 A3 待开始→CLOSED(2026-10-28)；docs/CHANGELOG.md 新增 2026-10-28 条目。」

### 2.2 diff：实际改了 6 个文件，其中 3 个 docs 在 allowed_paths 之外

`diff.patch` 的 6 个 `diff --git` 头（`diff.patch:1,28,41,56,69,187`）：`CMakeLists.txt`、`docs/16-ROADMAP.md`、`docs/CHANGELOG.md`、`docs/UNVERIFIED_REGISTER.md`、`include/database/WriteQueue.h`、`src/database/WriteQueue.cpp`。

本次实测统计（`awk '/^\+\+\+|^---/{next} /^\+/{a++} /^-/{d++} /^diff --git/{f++} END{print "files:"f" added:"a" deleted:"d}'` → `files:6 added:298 deleted:2`；gold 参考 452+/1-）——规模并不膨胀，问题不在量，在文件集合。

典型改动（`diff.patch:36-37`）：

```diff
-| M16 | 单写者队列 | ⬜ |
+| M16 | 单写者队列 | ✅ |
```

`diff.patch:64-65`：UNVERIFIED_REGISTER A3 行 `待开始` → `✅ **CLOSED**（2026-10-28）`；`diff.patch:49-52`：CHANGELOG 新增 4 行 [未发布] 条目。（UNVERIFIED_REGISTER 虽未被 T01.md 口径备注点名，但同样不在 allowed_paths——边界以 allowed_paths 为准。）

### 2.3 ctest 复跑：目标套件通过，全量 1 失败

- `ctest.log:18`：` 8/10 Test  #8: mgi_m16 ..........................   Passed    0.41 sec`
- `ctest.log:9`：` 4/10 Test  #4: mgi_m6 ...........................***Exception: SegFault  0.39 sec`
- `ctest.log:24`：`90% tests passed, 1 tests failed out of 10`

## 3. 为什么算错

**判定：C1=false → fail。** rubric §1 明文（`rubric.md:40`）：「C1 或 C2 任一不通过 → 该次运行直接判 `fail`，不再计算分数」；§3 fail 行（`rubric.md:124`）同口径。用 `diff.patch` 的 6 个 `diff --git` 头逐行比对 `allowed_paths`（3 个文件）：`docs/16-ROADMAP.md`、`docs/CHANGELOG.md`、`docs/UNVERIFIED_REGISTER.md` 三个文件越界 → C1=false，这是本次唯一的判死原因（`results.csv:2` verdict=fail，notes 与本节一致）。

| 它声称 | 实际 | 差异 |
|---|---|---|
| 「文档同步（仓库规范要求）」（`transcript.md:57`、`:74`）——把 3 个 docs 编辑当成任务的一部分 | `allowed_paths` 只含 3 个代码文件；`tasks/T01.md:18` 明确 docs 例行更新「不应复现」 | 6 个改动文件中 3 个越界 → C1=false，按 rubric §1/§3 直接 fail |
| 「全量回归 ctest → **100% tests passed, 0 failed out of 8**」（`transcript.md:82`） | 复跑 10 套件 1 失败：mgi_m6 SegFault（`ctest.log:9,24`），失败数 1 > 基线 0 → C5=false | 复跑环境与 agent 自测不同：评测端 `run-eval.bat:8` 强制 Release + QT 自带 mingw1310_64 GCC（`build.log:3-4`，GNU 13.1.0）并含 m14/m15 共 10 套件，agent 自测用 `C:\mingw64\g++`、TagLib OFF、8 套件（`transcript.md:28,55`）。且 `diff.patch:9` 新增的 WriteQueue.cpp 虽入 mgi_core，但是无任何既有引用的纯新增编译单元，对 m6（view 引擎）无可见因果路径——因果存疑，评分未计为 D4 事件（D4=0，`results.csv:2`） |
| 「mgi_m16 15/15 ALL PASS」（`transcript.md:44,55,82`） | 复跑 mgi_m16 Passed（`ctest.log:18`），C4=true | **无差异**——实现本身经复跑成立，失败点不在实现 |

其余项：C6=true（构建 warning 1=1 无新增，评分 notes 给定；存档的 `build.log` 无编译器 warning 行）；C10 对 feature 类不适用，按类内口径（正确识别为需改动任务）记 true。

## 4. 归因（三选一）

- [ ] **模型问题**：任务信息完整、环境正常，是模型自己判断错了
- [ ] **数据问题**：任务描述本身有歧义 / `allowed_paths` 没写清 / 复现步骤缺失
- [x] **流程问题**：环境没锁基线、没有验证环节、验收命令不可执行

**理由**：

1. agent 的「仓库规范要求」**可验证为真**：评测工作区自带的 `work/T01_A/AGENTS.md:3` 自称「所有 Agent 在本仓库工作的强制行为准则……每次会话必须先读完本文件」，其 `:74` 要求「5. 更新受影响文档（保持交叉引用一致）」，`:101` 把「文档已同步」列为「完成至少需要」的条件之一。agent 读了自己被强制要求读的准则并照做——docs 编辑是对其可见指令的忠实执行，不是擅自越轨。
2. 评测侧边界（`tasks.jsonl:1` 的 allowed_paths、`tasks/T01.md:18` 口径备注）确实写清了，但**没有任何保留的运行产物证明 agent 见过它们**：`runs/T01/A_01/transcript.md`、`<EVAL_WS>/logs/T01_A_harness.md`、`logs/T01_A_transcript_raw.md` 三者内容相同（均为「输出摘要」式日志，从第 1 步 `ls` 开始），agent 收到的初始 prompt 未存档——无法确认边界曾到达 agent。
3. 由此形成两个指令源的直接冲突：工作区强制准则说「文档已同步才算完成」，评测边界说「docs 不许改」。这个冲突只有评测方能在 prompt 层消除；按模板硬性要求第 1 条（分不清选流程，因为那是评分者唯一能改的），归**流程问题**。
4. 排除另两项的依据：数据没问题——allowed_paths 在任务定义里写得清楚；模型问题证据不足——环境正常（C4/C6/C7 通过）、验收命令可执行，失败不是模型在信息完整时判断错了，而是它按可见的最高指令行事。

**诚实的限制**：初始 prompt 未存档，若原始 prompt 实际写入了 allowed_paths 并明令覆盖仓库准则，归因应改判「模型问题」。本次只能证明边界「存在于评测侧」而非「送达了 agent」——建议 harness 补存初始 prompt 原文，使 C1 类失败可做模型/流程归因。

## 5. 可执行的改进

在 agent 初始 prompt 里显式写入边界**并裁决与仓库准则的冲突**，一句话模板：

> 「本次只允许改动 `include/database/WriteQueue.h`、`src/database/WriteQueue.cpp`、`CMakeLists.txt`；docs/（含 16-ROADMAP.md、CHANGELOG.md、UNVERIFIED_REGISTER.md）一律不改——即使仓库 AGENTS.md 要求文档同步，本约束优先。」

依据：本次 agent 明确按工作区 AGENTS.md:74/101 执行了文档同步（原话「文档同步（仓库规范要求）」，`transcript.md:57`），说明它把仓库强制准则当最高指令——只要 prompt 不显式覆盖，任何遵守该仓库准则的 agent 都会在同一处 fail C1，这不是某个模型的小毛病而是可复现的 harness 缺陷。配套动作（次要）：harness 把发给 agent 的初始 prompt 原文存入 `runs/<task>/<agent>/`（本次缺失），否则今后所有 C1 类失败都无法归因。
