# 任务候选池：查过但**没有**收录的套件

> 用途：记录"为什么某些套件没进任务集"。**不是待办清单，是排除记录。**
> 依据：仓库硬规则 §4.3「不许编 commit。拿不到真实 commit 就换任务，不要凑。」
> 核查日期：2026-10-24｜方法：在 MGI 仓库执行 `git log -S"<target>" -- CMakeLists.txt` 定位首次引入该测试目标的提交，再 `git show --name-only` 看它到底改了什么。

---

## 一、结论表

| 套件 | 断言 | 首次引入提交 | 结论 | 原因 |
|---|---|---|---|---|
| `mgi_m8` | 36 | `d45eafc` | **已收录为 T16** | 有真实实现（ProviderService / Providers / WinHttpTransport / IndexDatabase） |
| `mgi_m21` | 15 | `9be39c2` | **不收** | 该提交**只加了测试**（+ CMake + 文档），src 一行未改 |
| `mgi_m22` | 17 | `16877a4` | **不收** | 该提交只有一条 SQL 迁移 + 测试，域模型实现不在其中 |
| `mgi_m14` | 18 | `6c5cb8d` | **不收** | 实现在提交里，但**同时 vendor 了整套 TagLib**（100+ 文件）；基线没有它，Agent 无法凭空补出依赖 |
| `mgi_m28` | 10 | `41a6f79` | **不收** | 与 T07（`mgi_m27`）**同一个 commit**；一条任务不能挂两个 gold |
| `mgi_trem` | 26 CHECK | — | **不收** | 是 T2/T8 的补救回归，没有对应"用户可感知的功能"；且不在终报表内 |
| `mgi_gui` | 44 | — | **不收** | 验收项需要**人工点界面**，没有自动化判据；rubric 是机器证据 + 对照证据，判不了 |

**当前覆盖率**：19 条任务落在 MGI **16 个套件**（22 个中）+ KyoKiNexus 全套。
**未覆盖**：`m21` / `m22` / `m14` / `m28` / `trem` / `gui` 共 6 个。缺口原因是**证据不足**，不是遗漏。

---

## 二、逐条证据

### `mgi_m21`（统一外部身份验证，15/15）

```
$ git log -S"mgi_m21" --format=%H -- CMakeLists.txt | tail -1
9be39c280df8201831878a50da8ebf0d4cf6c338
$ git show --pretty="" --name-only 9be39c2
CMakeLists.txt
docs/16-ROADMAP.md
docs/CHANGELOG.md
tests/test_m21_identity.cpp
```

**为什么不收**：把 `tests/` 划掉之后，`allowed_paths` 只剩 `CMakeLists.txt` 一个文件。这意味着"任务"的正确答案几乎是空的——**被测对象在基线里就已经实现了**，测试一跑就过。这不是评测任务，是一条伪装的负控制。

> 如果硬要做这条，正确做法是拿它的**上游实现提交**当 gold。但上游是多个里程碑累积的，拿不到单一 commit，按 §4.3 只能放弃。

### `mgi_m22`（完整 Anime 域，17/17）

```
$ git show --pretty="" --name-only 16877a4
CMakeLists.txt
docs/16-ROADMAP.md
docs/CHANGELOG.md
scripts/_fix_bgm.py
src/database/migrations/005_anime_relations.sql
tests/test_m22_anime.cpp
```

**为什么不收**：`allowed_paths` = `CMakeLists.txt` + 一条 87 行的 SQL 迁移。任务是"实现完整 Anime 域（character / staff / voiced_by + 跨域验证）"，但 gold 只写了建表语句——**实体读写、关系查询的实现不在这条提交里**。把这条放进去，等于让评分者按 17 条断言去判一个"只加了表"的 diff。

### `mgi_m14`（TagLib 六格式实测，18/18）

```
$ git show --pretty="" --name-only 6c5cb8d | wc -l
   121
```

**为什么不收**：121 个文件里，`third_party/taglib/` 占了绝大多数（头文件 + `libtag.a` + CMake config）。真实改动只有 `MetadataReader.h/.cpp`、`IndexDatabase`、`ScanService` 四个文件，但**基线（`6c5cb8d^`）没有这套依赖**——Agent 要在基线里完成这条任务，必须自己把 TagLib 编译进来，这已经超出"改代码"的范围，变成"配环境"。

> 附带一个事实修正：`docs/UNVERIFIED_REGISTER.md` 里"TagLib 未装"那一条状态是 **CLOSED（A1）**——`mgi_m14` 的 18 条断言就是在 vendored TagLib 上跑通的。**再对外说"TagLib 没验"是错的。**

### `mgi_m28`（Plugin v2，10/10）

```
$ git log -S"mgi_m28" --format=%H -- CMakeLists.txt | tail -1
41a6f79...   # 与 T07（mgi_m27）同一个提交
```

**为什么不收**：一条任务对应一个 gold commit，`41a6f79` 已经挂在 T07 上了。两条任务共用一个 gold，会让"diff 是否落在 allowed_paths 内"这个判据失去意义——评分者分不清这次改动是为 m27 还是 m28。

### `mgi_trem` / `mgi_gui`

- `mgi_trem`（`test_t2_t8_remediation.cpp`，26 个 `CHECK`）是 T2/T8 问题的**补救回归**，没有独立的用户可感知功能，也没有出现在 `FINAL_TEST_REPORT.md` 的表里。它存在的意义是"防止旧问题复发"，不是"实现一个需求"。
- `mgi_gui`（44 项）来自 `USER_ACCEPTANCE §1–§5`，需要**人工点界面**。本仓库的 rubric 只用两类依据——机器证据（diff / ctest 退出码 / transcript）和对照证据（`allowed_paths`）。人工验收项两类都不是，判不了。**这条缺口要写进 README 的已知局限，不要假装覆盖了。**
- **口径调和**（与根 README §1/§5 对齐）：MGI 基线 `CMakeLists.txt` 里 `add_test(NAME mgi_gui COMMAND mgi_tests_gui)` 确实是 22 个 ctest 入口之一，offscreen 入口可自动跑（属 README 所说 22/22 之内）；但那 44 项验收判据本身需要人工点界面，rubric 的两类证据都判不了，故仍不收录为任务。

---

## 三、如果以后要补，需要什么

| 想补的 | 前置条件 |
|---|---|
| m21 / m22 | 找到**单一**上游实现提交（`git log --follow` 追到引入实现的那一次）；找不到就不做 |
| m14 | 先把 TagLib 依赖挪进基线（另开一条只看 `MetadataReader` 的任务，依赖预先就位） |
| m28 | 拆出独立的 Plugin v2 提交，或改用 m28 独有断言的子集 |
| gui | 需要先有可自动化的 UI 判据（截图基线 / 可编程驱动）。当前没有，不做 |

> **纪律**：以上任何一条都不许"先收进来，以后再补证据"。这个仓库的作废条件是"数字不可核"，不是"任务数不够"。
