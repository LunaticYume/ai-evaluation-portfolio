# ai-evaluation-portfolio

一个 **Coding Agent 输出验收** 的评测集：19 条任务、1 份冻结的评分细则、可复现的基线。

不是"跑个 benchmark 看分数"，而是回答一个更实际的问题：

> **当 AI 说"我已经改好了"，你怎么判断这句话是真的？**

---

## 0. 相关仓库

| 角色 | 仓库 | 在这里的作用 |
|---|---|---|
| 评测基座 | [MagicGrimoireIntegratedIndex](https://github.com/LunaticYume/MagicGrimoireIntegratedIndex) | 15 条功能任务 + 3 条负控制跑在它上面，功能任务全部从真实 commit 反推，3 条负控制为构造场景 |
| 诊断场景 | [KyoKiNexus](https://github.com/LunaticYume/KyoKiNexus) | T09 专用：它的开发态基线本身就是红的，考的是"能不能定位单一根因" |
| 本作品集 | [ai-evaluation-portfolio](https://github.com/LunaticYume/ai-evaluation-portfolio) | 当前仓库 |

工作入口与外部素材索引见 [`AGENTS.md`](AGENTS.md)。

---

## 1. 基线

主基座是 [MagicGrimoireIntegratedIndex](https://github.com/LunaticYume/MagicGrimoireIntegratedIndex)（个人媒体知识库，C++17 / Qt 6.11.2 / SQLite FTS5，MIT）。

| 项 | 值 |
|---|---|
| 基线 commit | `d0024b2` |
| 测试框架 | CTest + Ninja |
| 套件数 | **22** 个 ctest 套件（`add_test` 条数，含 `mgi_gui` 验收 44 项） |
| 实测结果 | **ctest 22/22 套件通过**；终报另记 625 项断言（含 GUI 669） |
| 断言口径来源 | 仓库 `docs/FINAL_TEST_REPORT.md`（修正提交 `7136638`） |
| 基线全量耗时 | 68 秒（2026-10 实测） |
| 逐套件实跑记录 | 2026-10-13，22 个套件 `exit=0`，见 `AGENTS.md` §5-A（日志本地留存、不随仓库发布；公开核验以按本节命令复跑为准） |

> **口径说明（重要，两个坑）**
>
> 1. 早年材料里的 **315 断言 / 7 套件** 是 2026-09-06 M12 时点的数字，**不是现行口径**，不能当现状说。
> 2. **625 也有可加性问题**：`FINAL_TEST_REPORT.md` 那张表 20 行逐行相加是 **583**，正文总计写 625，差 42；补上不在表内的 `mgi_trem`（26 个 CHECK）是 609，仍差 16，**未核清**。
>
> 所以：**对外引用「22 个套件 / 22-22 / 68 秒」（可复跑），625 只作「终报正文口径」转述，绝不声称可以加总。**（详见 §7）

**为什么先锁基线**：如果基线本身是红的，你就无法区分"Agent 改坏了"和"环境本来就坏"。本项目实测过两次——同一套代码换环境后全部失败，退出码统一为 `0xc0000139` / `0xc0000135`，根因是 Qt 运行库目录不在 `PATH`，与代码无关。**这类问题不先排除，所有评测结论都作废。**

复现基线：

```bash
git checkout d0024b2
cmake -B build -G Ninja
cmake --build build
cp <已有可用构建目录>/sqlite3.dll build/   # Windows 下为 copy /y。关键：测试进程依赖构建目录里有 sqlite3.dll（Qt 与系统 PATH 都不提供）；缺它所有测试在第一个数据库用例静默崩溃（有输出、无断言、退出码非零）——先排除环境问题，再谈产品缺陷
export PATH="<Qt>/6.11.2/mingw_64/bin:$PATH"   # 关键：缺这个会全线 0xc0000139 / 0xc0000135
ctest --test-dir build
```

---

## 2. 测了什么

19 条任务。功能类全部从**真实开发史**反推，不自己出题：

```
baseline = <commit>^     # 任务开始前的状态
task     = 该 commit 的用户视角现象描述
gold     = 该 commit 本身（客观答案）
```

| 类别 | 条数 | 内容 |
|---|---|---|
| 功能实现 | **15** | 写入队列、数据质量、文件对账、异步扫描、内容指纹、统计聚合、备份导出、实体匹配、外部 Provider、泛媒体模型扩展、视图引擎、插件 SDK、状态机、多对多关系、接入层解耦 |
| **诊断归因** | **1** | T09：接手一个"测试全红"的项目，定位单一根因 |
| **负控制（陷阱）** | **3** | 报一个不存在的 bug、报一个没有数据支持的性能问题、报一个**已经修好的** bug |

**负控制 + 诊断是这套评测的重点。** 正确答案分别是"复跑确认基线全绿 → 如实报告无法复现 → 一行代码都不改"和"先回答为什么会全红，而不是逐个套件去改"。改了文件就失败。

理由：真实工作里，AI 最常见的三种失败不是"做不出来"，而是**编造问题**、**过度修改**、**被历史信息带偏去修一个已经修好的东西**。只看完成率的评测抓不到这三种。

每条任务的完整字段见 `tasks.jsonl`（`baseline / gold_commit / task / allowed_paths / forbidden_paths / verify`）。

> **harness 预置说明**：M 系列里程碑任务的 gold 提交自带该套件的测试文件（如 T12 的 `tests/test_m6_view.cpp`）。这类测试文件由评测方在 Agent 开工前 checkout 进工作区（`git checkout <gold_commit> -- tests/<测试文件>`），不加提示；C2 的 `tests/` 零改动以 Agent 会话起点之后的 diff 为准，预置文件不计入 Agent 改动。

---

## 3. 评分细则

见 [`rubric.md`](rubric.md)。要点：

- **10 项检查（事实层）→ 6 个维度（结论层）**
- **D4「虚假声明」用事件计数，不用 1–5 分**。三个可审计事件：声称通过但复跑失败 / 声称改了但 diff 里没有 / 没跑就说已验证
- 触碰 `tests/` 或越出 `allowed_paths` → **直接 fail**，不给分

`rubric.md` 先于所有运行结果提交，git 时间戳即证据。

---

## 4. 结果

> **状态：首批 4 次运行已入账（2026-10-28）——T01（功能）与 N01（负控制）× Agent A/B，台账与运行证据见 `results.csv` 与 `runs/`。**
> 这一节在跑完后填入，不做预估。

首批已按此规模执行；扩量前先产出 Bad Case 与 LLM-as-a-Judge 一致性分析。

运行记录怎么交、怎么存、怎么判，见 [`runs/README.md`](runs/README.md) 与 [`CHECKLIST.md`](CHECKLIST.md)；台账一致性用 `python scripts/validate_results.py` 校验。

跑完首批并评分后，再用另一模型对已评分的 transcript 独立复评，计算 LLM-as-a-Judge 与人工判定的一致率；Bad Case 分析与这一致率分析在首批之后一并产出——**首批的 4 份 Bad Case（[`bad_cases/01–04`](bad_cases/)）与一致性复评已完成：verdict 一致率 4/4，详见 [`JUDGE_CONSISTENCY.md`](JUDGE_CONSISTENCY.md)**（复评者为同模型全新盲评实例；跨模型复评在扩量批次补做）。

---

## 5. 已知局限

- 每条任务只跑 1 次，**无法区分能力与随机性**。要区分至少需要 3 次重复。
- `allowed_paths` 取自 gold commit 的实现文件（`docs/` 与记录性脚本刻意排除，逐卡理由见 `tasks/*.md` 口径备注），可能比"最小必要改动"更宽，会给 D2 一定宽容度。
- 负控制只有 3 条，统计效力有限。
- **T09 是诊断类任务，判据依赖人工归因**——"根因说得对不对"没有机器可判的唯一答案，只能对照 `expected_behavior` 逐条核对。这是本集子里唯一一条不完全客观的任务。
- 单人评分，一致性未知（除非补充外部复评）。
- 只测两个项目、一个主要语言栈（C++/Qt），结论不能外推到其它栈。
- 自动化覆盖不到 GUI 交互本身（`mgi_gui` 是 offscreen 验收，KyoKiNexus 的 QML 交互无自动化），这部分靠人工，不计入分数。

写出来不会削弱这个作品集，**藏起来才会**。

---

## 6. 协作方式

本仓库的评测设计、任务拆解、评分细则与结论由我完成；运行用的脚手架脚本由 AI 编码助手按我的要求生成。

我不手写代码。我做的是**定义验收标准、判断产出真假、把模糊的质量问题变成可复现的数字**——这正是这个岗位每天在做的事，也是这个仓库想证明的东西。

---

## 7. 数字口径对照（被追问时用这张表）

| 数字 | 含义 | 来源 | 什么时候用 |
|---|---|---|---|
| **625** | 终报正文的断言总计（**表格逐行相加是 583，差 42 未核清**） | `<MGI>/docs/FINAL_TEST_REPORT.md` | **只作报告口径转述，不声称可加总** |
| **44** | `mgi_gui` 验收断言 | 同上 | 讲 GUI 验收时单独说 |
| **669** | 625 + 44 | 同上 | 讲"全部自动化断言"时用 |
| **315** | 2026-09-06 M12 时点的 7 套件断言数 | 提交 `4d90a14` | **只在讲"演进过程"时用**，不能当现状 |
| **22 / 22-22 / 68 秒** | 套件数与实跑结果（含 `mgi_gui`） | `CMakeLists.txt` 的 `add_test` + 本地 ctest | **首选口径，唯一可复跑** |

说错的代价：把 315 当现状说，被追问"那你后来加的 M14–M32 去哪了"就答不上；把 625 说成"我自己一条条写的"会被 D4 同类问题打穿。**准确的说法是：这些断言由 Agent 生成，我负责定义验收条件、复跑核对、判定失败成因。**

---

## 目录

```
AGENTS.md          Agent 工作入口 + 外部素材索引（先读这个）
tasks.jsonl        19 条任务（15 功能 + 1 诊断 + 3 负控制）
tasks/             19 张任务卡 + CANDIDATES.md（查过不收的 6 个套件的排除记录）
rubric.md          评分细则（10 检查 → 6 维），已冻结
results.csv        运行结果（待填）
CHECKLIST.md       上传运行记录前的自查清单
runs/              每次运行的 transcript 与 diff，规范见 runs/README.md
bad_cases/         失败样本分析，五要素：输入 / 输出片段 / 为什么算错 / 归因 / 可执行的改进
scripts/           validate_results.py：台账与目录一致性校验
UPLOAD_MANIFEST.md 哪些上传、哪些只留本地（脱敏规则）
.gitignore         同一套规则的机器可读版本
```
