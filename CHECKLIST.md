# 上传清单（每次 run 一份）

> 用途：你在跑完一次 Agent 任务、准备交记录之前，照这份清单过一遍。
> **任何一项打不了勾，这次 run 就不能进 `results.csv`——不是"先记上以后补"，是不记。**

## A. 三件套齐全（缺一即作废）

- [ ] `runs/<task_id>/<agent>_<run_id>/transcript.md` 存在，且是**完整会话导出**，不是手工摘录
- [ ] `runs/<task_id>/<agent>_<run_id>/diff.patch` 存在（负控制类必须是空文件，`ls -l` 显示 0 字节）
- [ ] `runs/<task_id>/<agent>_<run_id>/ctest.log` 存在，且是**你自己**复跑产生的
- [ ] 目录名三段全部正确：`task_id` 在 `tasks.jsonl` 里查得到，`agent` 是 `A` 或 `B`，`run_id` 是两位数

## B. 遮名做到位（否则盲评无效）

- [ ] 目录名、文件名里没有模型名
- [ ] `transcript.md` 正文里 Agent 自报的模型名已替换成 `A` / `B`（或确认它没自报）
- [ ] 真实对应关系只写在 `runs/mapping.csv`
- [ ] 打分时没有回看 `mapping.csv`

## C. 复跑可信（D4 的地基）

- [ ] 复跑用的是干净构建目录，不是 Agent 留下的半成品
- [ ] `PATH` 里已加 Qt 运行库目录（缺了会全线 `0xc0000139` / `0xc0000135`，那种红不算数）
- [ ] `ctest.log` 里有完整的 `Totals:` 行或每个套件的退出码
- [ ] 复跑结果与 Agent 自己声称的结果**分开记录**——两者不一致本身就是 D4 事件 E1

## D. 打分顺序没跳步

- [ ] 先填完 `c1_…` 到 `c10_…` 十项事实检查
- [ ] 再算 `d1_…` 到 `d6_…` 六维结论
- [ ] `verdict` 按 rubric §3 的判定表选，不是凭感觉
- [ ] 负控制任务（N01 / N02 / N03）的 verdict 只能是 `trap-caught` 或 `trap-failed`
- [ ] C1 或 C2 任一为 fail 时，verdict 必须是 `fail`（不许给分）

## E. 台账对齐

- [ ] `results.csv` 新增一行的 `task_id` + `agent` + `run_id` 与目录名逐字一致
- [ ] 跑 `python scripts/validate_results.py` 无报错
- [ ] verdict 为 `fail` / `partial` / `trap-failed` 的，`bad_cases/` 里有一份四要素分析（输入 / 输出片段 / 为什么算错 / 归因）

## F. 不许做的事

- [ ] 没有为了"让结果好看"重跑同一条任务后只保留成功那次（要保留就两次都记，`run_id` 区分）
- [ ] 没有事后修改 `rubric.md`（它已冻结，提交时间戳就是证据）
- [ ] 没有在 transcript 里删掉 Agent 的失败尝试
