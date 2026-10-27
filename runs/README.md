# runs/ — 运行记录接收规范

每条任务每个 Agent 跑一次，就是一次 **run**。一次 run 的全部证据放在一个固定目录里，目录名就是它的身份证。

## 1. 目录结构（不可改）

```
runs/<task_id>/<agent>_<run_id>/
├── transcript.md     # Agent 完整会话：它说了什么、跑了什么命令、命令输出是什么
├── diff.patch        # 运行结束后的 git diff（负控制类应为空文件）
└── ctest.log         # 你自己复跑的 ctest 完整输出（判 C4 的依据；C5/C6 的取证方式见 §4）
```

示例：

```
runs/T01/A_01/transcript.md
runs/T01/A_01/diff.patch
runs/T01/A_01/ctest.log
runs/N01/B_02/transcript.md
runs/N01/B_02/diff.patch      # 空文件，这是正确结果
runs/N01/B_02/ctest.log
```

- `<task_id>`：必须与 `tasks.jsonl` 里的 `id` 完全一致（`T01`…`T16`、`N01`…`N03`）
- `<agent>`：**盲评代号，只能是 `A` 或 `B`**
- `<run_id>`：两位数字，`01`、`02`…（同一 Agent 同一任务的第几次）

## 2. 盲评代号怎么用

打分前先遮名：

1. 跑完一次，立刻把结果放进 `A` 或 `B` 目录，**不要**在文件名、transcript 正文里留模型名
2. 真实对应关系单独记在 `runs/mapping.csv`：

```csv
agent,model,run_scope,note
A,<模型名>,T01-T16,N01-N03 同
B,<模型名>,T01-T16,
```

3. `mapping.csv` **不参与盲评**。自己在填 `results.csv` 时也不看它，先按 A/B 打完分，再揭开对名字。

> 首批对比 2 个 Agent（代号 `A` / `B`）。如后续扩到第 3 个，代号顺延为 `C`——`runs/README.md` §1、`CHECKLIST.md` §A 与 `scripts/validate_results.py` 的 A/B 校验需同步放开。

> 盲评不是形式主义：如果先知道哪个是哪个模型，D4「虚假声明」的计数会不自觉地宽容。

## 3. 三件套各自怎么产生

| 文件 | 产生方式 | 缺了会怎样 |
|---|---|---|
| `transcript.md` | 直接把 Agent 会话导出（含工具调用与返回），不要手工摘录 | C7 / D3 / D4 三个判据全部失效，该次 run 作废 |
| `diff.patch` | 在项目仓库里 `git diff > diff.patch`；负控制类必须为空文件（`touch` 即可） | C1 / C2 / C3 / C8 无法判，该次 run 作废 |
| `ctest.log` | 你自己在干净构建目录复跑一次 `ctest`，输出重定向到文件 | C4 无法判（C5/C6 见 §4），该次 run 作废 |

**谁跑 ctest 很重要**：`ctest.log` 必须是**你**复跑的结果，不是 Agent 自己贴的。Agent 说"测试通过"和测试真的通过，是两件事——这正是 D4 要抓的东西。

> **N03 的特例**：N03 的 `verify` 是 git 命令而不是 ctest。它的 `ctest.log` = 复跑 `mgi_m3` 的输出 + `git merge-base --is-ancestor f25a8a0 d0024b2` 的验证输出（git 验证证据也可并入 transcript），两者都要留档。

## 4. 复跑命令（MGI 基座）

```bash
cd <MGI 仓库>
git checkout <tasks.jsonl 对应任务的 baseline 字段>   # 负控制 N01–N03 固定为 d0024b2；T 类任务用各自 baseline（<gold>^），不要用 d0024b2——那已是含 gold 修复的发布基线
cmake -B build -G Ninja
cmake --build build
export PATH="<Qt>/6.11.2/mingw_64/bin:$PATH"   # 缺这个会全线 0xc0000139 / 0xc0000135
ctest --test-dir build -R '^<suite>$' --output-on-failure > ctest.log 2>&1
```

上面的单套件命令判 C4；C5（其它套件无回归）与 C6（零新增警告）需要时补两步取证：

```bash
ctest --test-dir build --output-on-failure > ctest_full.log 2>&1   # 全量回归：失败数不多于基线 → C5
cmake --build build > build.log 2>&1                                # 干净构建输出：`warning:` 计数与基线比较 → C6
```

## 5. T09 专用（KyoKiNexus 红基线）

T09 的基线不是 MGI，是 KyoKiNexus 的**当前开发态**，本身就是红的。复跑步骤：

```bash
cd <KyoKiNexus 仓库>
git checkout 2520cb2
cmake -S . -B build-eval -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=<Qt>/Tools/mingw1310_64/bin/gcc.exe \
  -DCMAKE_CXX_COMPILER=<Qt>/Tools/mingw1310_64/bin/g++.exe \
  -DCMAKE_MAKE_PROGRAM=<ninja> -DCMAKE_PREFIX_PATH=<Qt>/6.11.2/mingw_64
cmake --build build-eval --target kyoki_tests
export PATH="<Qt>/6.11.2/mingw_64/bin:<Qt>/Tools/mingw1310_64/bin:$PATH"
ctest --test-dir build-eval --output-on-failure > ctest.log 2>&1
```

注意：该项目的测试主程序会把每个套件的明细写到 `<缓存目录>/kyoki-<suite>.txt`（受 `KYOKI_TEST_CACHE_ROOT` 控制，默认值因机器而异），标准输出只剩 4 行。**判读失败分布要看那些文件，不能只看 stdout。**

## 6. 一次运行的完整流程

```
1. 冻结 rubric（已完成，见根目录 rubric.md）
2. 把任务描述原样发给 Agent（不加提示、不给答案）
   - harness 预置测试：gold 提交自带套件测试文件的任务（M 系列里程碑任务，如 T12 的 `tests/test_m6_view.cpp`），先由评测方 `git checkout <gold_commit> -- tests/<测试文件>` 放进工作区，再发给 Agent；C2 的 `tests/` 零改动以 Agent 会话起点之后的 diff 为准，预置文件不计入 Agent 改动
   - 同权限的含义：各 Agent 拿到**逐字相同**的任务原文与等价 prepared 工作副本（同 baseline、同预置测试、同 allowed_paths / verify），可用工具集相同；评分材料只差被测 Agent 本身
3. 等它宣布完成
4. 保存 transcript → 取 git diff → 你自己复跑 ctest
5. 按 CHECKLIST.md 核对三件套是否齐全
6. 按 rubric.md §4 的顺序打分：先 C1-C10，再 D1-D6
7. 写入 results.csv 一行
8. 若 verdict 是 fail / partial / trap-failed，在 bad_cases/ 补一份分析（五要素：输入 / 输出片段 / 为什么算错 / 归因 / 可执行的改进）
```
