# AGENTS.md — 本仓库的 Agent 工作入口

> 任何 Agent（人或 AI）改动本仓库前先读这一份。
> 它回答三件事：**这仓库想证明什么、哪些材料是可信的、哪些动作会毁掉它。**

---

## 0. 这是什么

一个 **Coding Agent 输出验收** 的评测集：19 条任务、1 份冻结的评分细则、可复现的基线。

它不回答"哪个模型更聪明"，只回答一个更实际的问题：

> **当 AI 说"我已经改好了"，你怎么判断这句话是真的？**

## 1. 三个仓库链接

| 角色 | 链接 | 在这里的作用 |
|---|---|---|
| **评测基座** | [MagicGrimoireIntegratedIndex](https://github.com/LunaticYume/MagicGrimoireIntegratedIndex) | T01–T08、T10–T15、N01–N03 全部跑在它上面；任务从它的真实 commit 反推 |
| **诊断场景** | [KyoKiNexus](https://github.com/LunaticYume/KyoKiNexus) | T09 唯一使用。它的**当前开发态基线本身就是红的**，这正是 T09 要考的东西 |
| **本作品集** | [ai-evaluation-portfolio](https://github.com/LunaticYume/ai-evaluation-portfolio) | 就是你现在读的仓库 |

本地副本位置（只读，**不要**往里面提交本仓库的东西）：

```
<本机只读副本路径已隐去——本仓库对外发布，不在公开文件里暴露本地目录结构>
```

> ⚠️ 注意：硬盘上另有几份同名目录是**不可用副本**（一份只剩 docs 且零 ref，另一份是 Electron 版）。用之前先确认 `git log` 与源码树，别拿错。

## 2. 目录与职责

```
tasks.jsonl     19 条任务（15 功能 / 1 诊断 / 3 负控制），字段见下方 §3
rubric.md       评分细则（10 检查 C1–C10 → 6 维 D1–D6）。**已冻结，不许改**
results.csv     运行台账（22 列）。当前只有表头
runs/           每次运行的证据，目录规范见 runs/README.md
bad_cases/      失败样本分析，四要素：输入 / 输出片段 / 为什么算错 / 归因
scripts/        validate_results.py：校验台账与目录一致性
CHECKLIST.md    上传一次运行记录前的自查清单
```

## 3. 任务的字段约定

功能类任务（`category: feature`）必须有：

`id / category / track / baseline / gold_commit / suite / gold_assertions / task / allowed_paths / forbidden_paths / verify / source / difficulty`

诊断类与负控制类（`diagnosis` / `negative-control`）额外要求 `expected_behavior`，且 `allowed_paths` 为 `[]`、`forbidden_paths` 为 `["*"]`——**这类任务的正确答案是不改代码**。

`baseline` 与 `gold_commit` 必须是真实存在的 hash。**写进去之前必须用 `git log -1 <hash>` 验过**，验不过就不写。

## 4. 硬规则（违反＝这个仓库作废）

1. **`rubric.md` 已冻结。** 它的提交时间早于任何运行结果，这是"标准先于结果"的唯一证据。要改必须单独提交并写明原因。
2. **数字必须可核。** 每个数字都要能指回 `tasks.jsonl` 的 `gold_assertions`、commit message，或项目 `docs/` 里的报告。
3. **不许编 commit。** 拿不到真实 commit 就换任务，不要凑。
4. **不许用改测试的方式让测试通过。** `forbidden_paths` 一律含 `tests/`；C2 不过直接 fail，没有讨论空间。
5. **负控制与诊断类任务，改文件就是失败。** 这是整个集子里最值钱的三条（N01/N02/N03）+ 一条（T09）。
6. **不许在结果出来之后再调 rubric 或删任务。** 跑砸了也留着，局限写进 README §5。

## 5. 有效素材索引（本地材料，**不随本仓库发布**）

> 这些是散落在本机的测试 / 评测材料，**只用于校准口径，不上传**。具体路径已隐去；引用前先按 §4 核对可信度。**按可信度排序**：

### A. 实跑证据（最硬，可直接引用）

| 材料 | 说明与口径 |
|---|---|
| MGI 22 套件实跑记录 | 2026-10-13，逐套件 `exit=0` + 耗时，22/22 PASS；同目录留有原始输出日志 |
| MGI 夹具排错记录 | m14/m18/m_gui 首轮 `exit=3`（`cannot copy file: bad_apple.flac`）→ 工作目录切到 `build` 后全通过。**这是"把测试失败归因为环境而非产品缺陷"的原始证据** |
| KyoKiNexus 实跑记录 | 2026-10-13，`exit=31` / FAIL；明确写了"README 的 100% passed 是 v1.0.1 发布基线，当前开发态不是" |
| KyoKiNexus 根因线索 | 记载 `datatype mismatch Unable to fetch row`（仓储初始化阶段）——**T09 的根因就在这句里** |

> ⚠️ **口径纠正**：早年报告写"22/22 与 315 断言一致"。**315 是 2026-09-06 M12 时点（7 套件）的数**，现行是 22 套件；断言总数见 README §1 的加总说明——**不要声称可加总**。

### B. 表述与训练材料（用来校准话术）

| 材料 | 用途 |
|---|---|
| 核心判断与表述红线 | 哪些话不能说、哪些能力要降级表述。**引用前必读** |
| 项目分工与训练定位 | 为什么选 KyoKiNexus 当诊断场景 |
| 测试训练模式提示词 | "Agent 不许替本人完成整个测试"的具体规则 |
| 背景与求职问答建议 | 专业 / 空档期 / 自我评价的处理口径 |

### C. 测试专项（面试追问用）

对应本仓库的：任务描述怎么写成"用户视角现象" / `allowed_paths` 与 `verify` 的思路来源 / 自动化覆盖不到的部分 / SQLite 断言与数据一致性 / 被追问"覆盖率多少"时怎么答。

### D. 知识与岗位校准

AI 产品测试与传统测试的差别（预期不唯一）；评测岗 JD 关键词（Benchmark / Golden Set / 评测任务集合 / 质量基线）；个人能力矩阵与薄弱点。

### E. 已被本仓库取代（不要再用）

早期版 `coding-agent-eval/`（README 746 B / rubric 1 655 B）已被本仓库取代，只作历史留档。两份同时存在会造成口径冲突。

## 6. 当前状态

| 项 | 值 |
|---|---|
| 任务数 | **19**（feature 15 / diagnosis 1 / negative-control 3） |
| 运行记录 | **0**（`results.csv` 只有表头） |
| 待办 | 等用户上传 Agent 运行记录（transcript / diff / ctest.log），按 `CHECKLIST.md` 核对后写入台账 |

## 7. 给 Agent 的操作建议

1. **先跑 `python scripts/validate_results.py`**，确认台账与目录没有漂移。
2. 改 `tasks.jsonl` 前先 `git log -1 <hash>` 验 commit；改完再跑一次校验脚本。
3. 引用任何数字前，回到 §5 确认它属于哪个口径（315 还是 625）。
4. 不要替用户"预填" `results.csv`。没有真实运行就没有行。
5. 不要动 `rubric.md`、不要动 `bad_cases/TEMPLATE.md` 的四要素结构。
