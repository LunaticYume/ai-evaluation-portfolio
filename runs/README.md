# runs/ — 运行记录接收规范

每条任务每个 Agent 跑一次，就是一次 **run**。一次 run 的全部证据放在一个固定目录里，目录名就是它的身份证。

## 1. 目录结构（不可改）

```
runs/<task_id>/<agent>_<run_id>/
├── transcript.md     # Agent 完整会话：它说了什么、跑了什么命令、命令输出是什么
├── diff.patch        # 运行结束后的 git diff（负控制类应为空文件）
└── ctest.log         # 你自己复跑的 ctest 完整输出（判 C4 / C5 的唯一依据）
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

- `<task_id>`：必须与 `tasks.jsonl` 里的 `id` 完全一致（`T01`…`T15`、`N01`…`N03`）
- `<agent>`：**盲评代号，只能是 `A` 或 `B`**
- `<run_id>`：两位数字，`01`、`02`…（同一 Agent 同一任务的第几次）

## 2. 盲评代号怎么用

打分前先遮名：

1. 跑完一次，立刻把结果放进 `A` 或 `B` 目录，**不要**在文件名、transcript 正文里留模型名
2. 真实对应关系单独记在 `runs/mapping.csv`：

```csv
agent,model,run_scope,note
A,<模型名>,T01-T15,N01-N03 同
B,<模型名>,T01-T15,
```

3. `mapping.csv` **不参与盲评**。自己在填 `results.csv` 时也不看它，先按 A/B 打完分，再揭开对名字。

> 盲评不是形式主义：如果先知道哪个是哪个模型，D4「虚假声明」的计数会不自觉地宽容。

## 3. 三件套各自怎么产生

| 文件 | 产生方式 | 缺了会怎样 |
|---|---|---|
| `transcript.md` | 直接把 Agent 会话导出（含工具调用与返回），不要手工摘录 | C7 / D3 / D4 三个判据全部失效，该次 run 作废 |
| `diff.patch` | 在项目仓库里 `git diff > diff.patch`；负控制类必须为空文件（`touch` 即可） | C1 / C2 / C3 / C8 无法判，该次 run 作废 |
| `ctest.log` | 你自己在干净构建目录复跑一次 `ctest`，输出重定向到文件 | C4 / C5 无法判，该次 run 作废 |

**谁跑 ctest 很重要**：`ctest.log` 必须是**你**复跑的结果，不是 Agent 自己贴的。Agent 说"测试通过"和测试真的通过，是两件事——这正是 D4 要抓的东西。

## 4. 复跑命令（MGI 基座）

```bash
cd <MGI 仓库>
git checkout d0024b2
cmake -B build -G Ninja
cmake --build build
export PATH="<Qt>/6.11.2/mingw_64/bin:$PATH"   # 缺这个会全线 0xc0000139 / 0xc0000135
ctest --test-dir build -R '^<suite>$' --output-on-failure > ctest.log 2>&1
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

注意：该项目的测试主程序会把每个套件的明细写到 `<CACHE_DIR>/kyoki-<suite>.txt`（受 `KYOKI_TEST_CACHE_ROOT` 控制），标准输出只剩 4 行。**判读失败分布要看那些文件，不能只看 stdout。**

## 6. 一次运行的完整流程

```
1. 冻结 rubric（已完成，见根目录 rubric.md）
2. 把任务描述原样发给 Agent（不加提示、不给答案）
3. 等它宣布完成
4. 保存 transcript → 取 git diff → 你自己复跑 ctest
5. 按 CHECKLIST.md 核对三件套是否齐全
6. 按 rubric.md §4 的顺序打分：先 C1-C10，再 D1-D6
7. 写入 results.csv 一行
8. 若 verdict 是 fail / partial，在 bad_cases/ 补一份分析（四要素）
```
