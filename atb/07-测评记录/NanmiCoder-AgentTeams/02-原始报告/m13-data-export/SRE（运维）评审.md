# m13-data-export SRE（运维）评审

> t18 · n46-sre · 2026-11-17

---

M13（用户数据导出）SRE 评审完成：结论「有阻塞项（4 P0 + 4 P1），不建议按现行方案上线」。报告：_work/评审-M13-数据导出-SRE.md。注意 atb/tasks/m13-data-export/review-perf.md 是性能视角，本报告只写运维面、不重复。

P0-1 「异步任务 + 落盘 + TTL」在本仓库三件基础设施都没有。① 任务类型是 CHECK 闭集（0001:347-348、0007:25-26、field-rules.ts:88-95，tests/migrate-transaction.test.ts:99-115 断言一致），没有 me_export 可用；放开 CHECK 要重建表，migrate.ts:97-115 明说这种迁移失败留半成品。② 即便放开，handlers.ts:369-377 没注册 → main.ts:91-104 判 no_handler + skipped（不重试）。③ 产物根不存在：compose.yaml:44-108 只挂 /app/.data 与 /app/backups，/data/exports 无卷；web.Dockerfile:52-55 只建 /app/.data 且 USER 10001 → mkdir /data 直接 EACCES；若改落 /app/.data/exports，它不在 media 根下，backup.mjs:99-103 不会打包 → 容器重建即丢。
P0-2 「TTL 7 天，定时清理」：本仓库明确否决过库自带 setInterval 清理（rate-limit.ts:182-184「要进程能被干净关掉」），且 worker 主循环 main.ts:170-206 无清扫、deploy/ 无清理 timer、grep DELETE FROM background 0 命中。后果：产物只增不减压在 /app/.data，而 health.ts:50,94 的磁盘阈值只有 200MB、超限即 /api/health 503（health/route.ts:30-58）→ 挤爆磁盘连带 SQLite；更严重，TTL 未删 = 超期保存个人数据，正好违反方案自述的合规目标。
P0-3 产物是个人数据全量副本，却零审计/零归属校验/备份语义冲突：项目所有写操作都走 operation_records（content/index.ts:345-359 是范例）；下载端点未要求校验 tasks.requested_by=ctx.userId，而 taskId 是 UUIDv7（field-rules.ts:14，含时间戳可枚举）；backup.mjs:62-70 全量热备 + BACKUP_KEEP=14（compose.yaml:62）→ 「用户已删除的数据」会继续活在 14 份历史备份里，方案无任何说明。
P0-4 方案代码块的 enforceRateLimit 照抄即崩：真实签名是 enforceRateLimit(db, action, userId, points, override)（rate-limit.ts:196-208），方案传了对象；'me_export' 不在 RateLimitAction 闭集（rate-limit.ts:46-55），RATE_LIMIT_RULES 是 Record 全套会缺字段（:68-134）。另外 assertCanWrite（context.ts:34-43）会挡禁用账户——禁用账户仍有权导出自己的数据，应改 isAuthenticated（context.ts:19-21）。

P1-5 内存/租约：better-sqlite3 同步驱动，.all() 一次性物化 + JSON.stringify 再来一份（仓库已有同类记录 n60:32）；租约 120s、截止 300s（constants.ts:19,45），超时被 recoverExpired（service.ts:402-476）置回 retry_wait → 两份导出写同一文件、半截 JSON；worker 单循环串行（main.ts:190-205）会堵住所有人的发布任务。
P1-6 §4 可见性过滤把个人历史抹掉：resolveVisibilityInTx 语义是「对外公开」（content/index.ts:153-166），撤回/隐藏/未发布的全部消失；且它对 forum_posts / plugin_favorites 根本不适用（查的是 content_entities）。
P1-7 范围与形状：只列 3 张表，缺 replies/comments/notifications/preferences/corrections/feedback/role_applications/drafts（migrate.ts:142-237 里用户相关的十几张）；ownerColumn+join 的元数据驱动查询要求实现者拼 SQL，仓库无此先例。
P1-8 可观测性：状态端点不说失败（service.ts:325-342 有 last_error 但用户看不到、diagnose.mjs:156-171 只列最近 10 条）；/api/health 不做业务统计（health.ts:1-9）→ 导出积压无告警。
P2-9 路径/容量：导出根须服务端生成（对齐 file-store.ts:42-80 铁律）；无体积上限，与媒体共享 /app/.data 卷（compose.yaml:65,92）；须按 docs/docker约定.md:12 明确宿主落点。

另附 7 项上线验收清单（类型可落地/产物不丢/到期真删/下载只给自己/审计有据/不撑爆内存与租约/用户看得懂失败）。