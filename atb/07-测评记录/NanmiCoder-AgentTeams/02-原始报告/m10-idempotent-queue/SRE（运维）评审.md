# m10-idempotent-queue SRE（运维）评审

> t9 · n46-sre · 2026-11-17

---

M10（幂等去重队列）SRE 评审完成：结论「有阻塞项，不建议按现行方案上线」。

报告落盘：_work/评审-M10-幂等队列-SRE.md

P0（阻塞）
1. §2 按 source 拆 dedup key = GitHub 配额被瓜分。未认证时核心接口 60/h，代码把单轮预算压到 58（discovery.ts:49-52,71-80；deploy/discovery-drip.sh:2-3 亦写明"一轮就吃满"）。N 个 source 同天跑 → 撞 403；更糟：收录阶段 provider_unavailable 是 throw（discovery.ts:290-297）→ handler 返回 error（handlers.ts:353-360）→ 队列按 [30s,120s] 重试 2 次（service.ts:331-341），每次重试再打一轮 3×20s（github-discovery.ts:153,178-219），必然再次 403 → 3 倍配额、0 条插件、任务 failed。建议：多 source 共享同一 maxRequests 且串行（现 main.ts:190-205 是并行领跑），或先上 GITHUB_TOKEN 并写死"未认证只允许 1 source/天"。
2. §3 手动触发语义错了：enqueue 命中已存在行直接返回原 taskId（service.ts:204-208），当天已 succeeded/failed 后管理员点"立即触发"拿到 202 + 旧 taskId，什么都没跑。应保留 manual:<source>:<requestKey>，防连点用限流而不是永久幂等。
3. §4 照抄 [30s,120s]：远小于小时级配额窗口，discovery.ts:170-173 自己的注释就写明重试改变不了配额。应区分类限流为 skipped/跨小时退避，并说明"一天一次的任务失败 3 次 = 当天没有发现"如何告警。

P1
4. 键格式切换无兼容计划：旧 daily: 与新 discovery: 会并存（UNIQUE 只做值域唯一，0001:369 / 0007:47），切换当天同轮跑两次；后台页 route.ts:29-67 原样展示两种 key；drip timer（每 40 分钟）去留未提。
5. 可观测性为零：失败任务不会写 operation_records（handlers.ts:318-351 只在成功路径），/api/health 不含任务指标（health.ts:56-147），diagnose.mjs:156-171 只列 recent_problems 不影响退出码 → 发现任务可以静静失败一整天没人知道；且 background_tasks 全仓无清理策略（grep DELETE FROM background 0 命中），按 source 拆后行数再乘 N。
6. 关不掉/回滚不了：现有刹车 DSH_REVIEW_CATALOG_OFFLINE / DSH_REVIEW_DISCOVERY_DAILY 只作用于排程（scheduled.ts:45-53），手动 API（route.ts:86-131）与 scripts/discover-now.mjs:78-81 都不看它们（discover-now 的 --dry-run 后直接跳过离线判断）；键格式改动没有回滚路径。

P2
7. 容量/索引：background_tasks 无 task_type 索引（0001:383-384），后台页 WHERE task_type=… ORDER BY created_at DESC（route.ts:29-37）会全表扫描+排序，随"每 source 每天一行且只增不删"持续变慢。建议加 idx_tasks_type_created。
8. 前提核实：time.ts 无 ymd（实现者最可能写 toISOString().slice(0,10) → UTC 日界，与 discoveryDailyKey 的 Asia/Shanghai 差 8 小时，tests/plugin-discovery.test.ts:585-589 已钉死）；方案 enqueue 代码块缺必填的 schedulingGroup / availableAt（service.ts:194-199 会抛 err.input，即每 60 秒抛一次）；payload 键名 {source,date} 与 handler 读的 run_key 不一致（handlers.ts:288-315，项目历史上踩过同样漂移）。

另附"上线前必须补齐的 6 项验收清单"（配额不被打穿 / 手动触发可用 / 关得掉 / 失败可见 / 回滚可行 / 时区一致）。

注：该方案要解决的问题本身在现状下基本不存在——自动路径每天只排 1 个任务（scheduled.ts:70-129），手动路径本来就是 manual:<requestKey>/时间戳的新任务（route.ts:108-113、discover-now.mjs:89），"同一天多次触发导致重复调 GitHub"这个背景不成立。