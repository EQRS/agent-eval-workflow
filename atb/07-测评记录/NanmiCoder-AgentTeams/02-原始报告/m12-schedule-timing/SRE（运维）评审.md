# m12-schedule-timing SRE（运维）评审

> t15 · n46-sre · 2026-11-17

---

M12（定时任务调度时间改造）SRE 评审完成：结论「有阻塞项（4 P0 + 4 P1），不建议按现行方案上线」。报告：_work/评审-M12-调度时间改造-SRE.md（另有前人对同一文档的 _work/评审-SRE-调度时间改造.md；本报告不重复其结论，补运维面）。

P0-1 停机不补跑且完全不可见（§1 加了 scheduled_at 却全篇不用；§2 用 ymd(now) 生成键 + now>=target 判断）。23:59 停机、次日 00:05 恢复 → 昨天那天的 key 永远没写过 → 该日永久漏跑、库里 0 行；diagnose.mjs:139-175 只统计已存在的行，health.ts 不含任务指标 → 运维与监控都看不见。且 03:00 与备份 timer（deploy/dsh-review-backup.timer:12，带 RandomizedDelaySec 5m）同分钟抢同一 SQLite 文件 IO。建议：引入"上次成功轮次日期"水位 + 只补 1 天（对齐架构 94.2，docs/项目规划/架构设计.md:2793）；scheduled_at 真正落库并按 lag 报警；错峰到 03:30。

P0-2 与既有 ensureDailyDiscoveryTask 并存 → 同一天排两条 plugin_discovery（main.ts:178-188 用 daily:<date>，方案用 discovery:github:<date>，UNIQUE 只做值域唯一 0001:369/0007:47）。main.ts:190-205 会一轮领满 4 个再串行执行，两条连跑；未认证 GitHub 核心配额 60/h、单轮预算 58（discovery.ts:49-52,71-80）→ 第二条必 403；更糟 discovery.ts:290-297 在收录阶段 throw → handlers.ts:353-360 返回 error → [30s,120s] 重试（service.ts:331-341），每次重试再打 3×20s 上游重试（github-discovery.ts:153-178）→ 约 3 倍配额 0 条插件、任务 failed。

P0-3 cache_warmup 无法落地：TaskTypes（field-rules.ts:88-95）与 DB CHECK 必须逐条一致（tests/migrate-transaction.test.ts:99-115 有断言）；handlers.ts:369-377 没注册 → main.ts:91-104 判 no_handler + skipped 终态（skipped 不重试）→ 上线即"每天 04:00 一条静默跳过"。迁移 0042 两种写法都有坑：普通迁移放不开 CHECK（migrate.ts:116-121）；-- dsh:migration-transaction=off 走 0007 式重建（migrate.ts:97-115 明说不回滚，失败留半成品），而 background_tasks 被 background_task_attempts.task_id(0001:388) 与 operation_records.task_id(0001:424) 引用，重建期间 enqueue/claim 会撞 no such table。另：docker/compose.yaml:27-42 的 migrate 是前置一次性容器、deploy/dsh-review-worker.service:8 只是 After=，都不能阻止旧进程持连接跑；且仓库迁移到 0041 为止，0042 未被占用但合并时易撞号。

P0-4 方案代码块照抄就崩：enqueue 缺必填的 schedulingGroup 与 availableAt（service.ts:194-199 抛 err.input）→ tick 每轮抛异常，被 try/catch 吞成日志行，任务永不入队而 worker 心跳照常（heartbeat.ts:132-151 独立定时器）→ 健康检查全绿、业务静默失效。payload {date} 与 handler 读的 run_key 不一致（handlers.ts:288-315），run_key 缺失回退 discoveryDailyKey(handlers.ts:292) → 审计键与任务脱节。

P1-5 时区未钉死：time.ts 无 ymd/atTime，最自然实现 toISOString 是 UTC 日界，与既有 Asia/Shanghai（discovery.ts:121-124、tests/plugin-discovery.test.ts:585-589）差 8 小时；北京 00:00-08:00 期间 today 退回旧日期 → 03:00 那轮直接不排。容器时区由 compose 钉死（compose.yaml:38,53,88），systemd 依赖宿主 localtime → 两种部署形态解释不同。
P1-6 scheduled_at 与 idx_tasks_scheduled 从未被查询使用（claim 只按 available_at，service.ts:116-179）→ 纯写放大；且 background_tasks 无 task_type 前缀索引，后台页 route.ts:29-37 全表扫描排序。
P1-7 关不掉/回滚不了：现有刹车 DSH_REVIEW_CATALOG_OFFLINE / DSH_REVIEW_DISCOVERY_DAILY（scheduled.ts:16-17,45-53）若不搬进新 tick 就失效，cache_warmup 无任何开关；SQLite DROP COLUMN 受限（ADD COLUMN 删不掉）；background_tasks 无保留期（grep DELETE FROM background 0 命中）。
P1-8 tick 节拍未定：IDLE_SLEEP_MS=10s（constants.ts:25,48）→ 一天 8640 次 × 2 schedule × enqueue 的 SELECT；丢了既有 60s 节流（scheduled.ts:28）。须写死"只在 runOnce 内调用、禁止独立 setInterval"。
P2-9 与架构 94.2 相反（架构设计.md:2793 要求短事务校验到期 + 唯一轮次 + 补一次当前刷新）。

另附 7 项上线验收清单（不漏跑/不重跑/关得掉/失败可见/迁移可回滚/时区一致/与备份错峰）。