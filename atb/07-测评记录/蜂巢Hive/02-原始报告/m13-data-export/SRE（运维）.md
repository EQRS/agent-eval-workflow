# m13-data-export · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_题库全测_验证单元_0012
> 模型 deepseek-chat · 6206 字 · 31 次工具调用

---

仅从 SRE（运维）角度评审完成。已核实方案、4 个相关迁移（0012/0019/0037/0039）、worker 队列、rate-limit、file-store、health 与 deploy 单元文件。

---

# SRE 评审：m13 用户数据导出

总结论：**方向对，运维面上不可上线**。方案解决了"怎么生成"，完全没写"谁清、清不掉怎么办、磁盘满了怎么办、worker 忙时新功能会不会饿死"。下面 10 条按阻塞程度排序。

---

## P0 — 上线前必须解决

### 1. 产物落 `/data/exports`：容器重启即丢，磁盘水位无人看管

**① 问题在哪**：proposal.md:57 直接写死绝对路径 `/data/exports/<userId>/<taskId>.json.gz`。

**② 后果**：
- 本仓实际约定是 `.data` 持久目录加环境变量覆盖（`src/server/platform/file-store.ts:161` 的 `defaultMediaRoot()` 用 `DSH_REVIEW_MEDIA_ROOT`，默认 `join(process.cwd(), '.data')`）。`/data` 既不在 `.env.example` 的白名单里，也不在 docker/deploy 的挂载点上——**容器重建后导出文件全部消失**，而任务表里状态还是 `succeeded`，用户拿到 404（`GET /me/export/:taskId/download`），投诉工单无从解释。
- 该路径不经过 `assertSafeStorageKey` / `resolveWithinRoot`（file-store.ts:44-83 的全套穿越防护），`<userId>` 是 UUID 尚可，但一旦有人图省事把邮箱或 slug 塞进路径，就是路径穿越入口。
- 健康检查完全看不到这块盘：`src/server/platform/health.ts:47` 的 `minFreeMb` 默认 200MB，且只 check `dbFile` 所在分区。导出文件把盘写满时，**SQLite WAL 写失败**——那是全站故障，而不是导出功能故障。

**③ 建议**：
- 路径改为 `<media root>/exports/<userId>/<taskId>.json.gz`，复用 `createFileStore` + `assertSafeStorageKey`，storage_key 写库（参照 0003 的附件模型）。
- 导出目录**单独挂卷**，并在 health 里加一项 `disk.exports_free`（独立阈值，如 2GB），低于阈值时拒绝新导出任务（返回 503 + 明确错误码），而不是让它写满库所在盘。

### 2. "TTL 7 天，定时清理" 是一句没有主语的话——仓库里没有可用定时器

**① 问题在哪**：proposal.md:60。

**② 后果**：仓库的定时机制**全部**是 systemd timer（`deploy/*.timer` 只有 `dsh-review-backup.timer`、`dsh-review-discovery-drip.timer`），而 worker 内置排程（`src/worker/scheduled.ts:1-20`）只排"每日插件发现"一种，且注释已明说其代价是"一整天没跑就跳过、不补历史"。所以按当前方案：
- 清理逻辑没人调用 → 文件永久堆积 → 第 1 条的磁盘打满变成**必然**，不是万一。
- 而且删文件与删任务行必须同源，否则出现"文件没了、行还在 succeeded"或反之。

**③ 建议**：明确写清清理载体。二选一并写进方案：
- **(推荐) 放进 worker 的 `scheduled.ts`**，按 `deduplication_key = export_cleanup:<yyyy-mm-dd>` 幂等排入，与 `ensureDailyDiscoveryTask` 同构，天然重启自愈；
- 或加 `deploy/dsh-review-export-cleanup.timer`（带 `Persistent=true`）。
清理必须**先删文件后删行**（反序会留下无法回收的孤儿文件），且失败要打 warn 日志让监控能抓。

### 3. 新增任务类型要过迁移闸门，方案里一字未提

**① 问题在哪**：proposal.md 通篇没提 `export` 任务类型与 `background_tasks` 的对接。

**② 后果**：`TaskTypes` 是硬编码注册集合，注释明确"注册集合随迁移版本维护，不能由配置注入新类型"（`src/packages/common/field-rules.ts:88-89`）。未注册的类型在 worker 里会被判 **skipped**（`src/worker/main.ts` 的 handler 注释："未注册的任务类型会被判为 skipped（不静默成功）"）。即：`POST /api/me/export` 返回 202，任务永远 skipped，用户看着"处理中"到天荒地老。这是**最典型的静默失败**。

**③ 建议**：方案补一节"迁移 0043"：`TaskTypes` 加 `user_export`（同时确认 `background_tasks.task_type` 的 CHECK 需不需要重建表——0001 baseline 里的 CHECK 是硬约束，加枚举值往往要重建），并加一条守卫测试（参照 `tests/plugin-discovery.test.ts` 的防漂移用例思路）断言 handler 注册表覆盖了所有 `TaskTypes`。

### 4. 4 个并发槽位 + 全局 FIFO：导出任务会饿死内容发布

**① 问题在哪**：proposal.md:1 的预期"不阻塞请求，不影响其他用户"，与实现机制矛盾。

**② 后果**：`MAX_CONCURRENT_TASKS = 4`（`src/server/modules/tasks/constants.ts:19`），`claimWithin` 是 `ORDER BY available_at ASC, created_at ASC, id ASC`（`src/server/modules/tasks/service.ts` 的 SELECT）——**严格全局 FIFO，没有优先级、没有按组的配额**。`schedulingGroup` 字段虽然存在且被写入，但 claim 的 SQL **完全不读它**。后果：
- 一个用户点 3 次导出（限流可能允许），就吃掉 4 个槽位中的 3 个；
- 队列里已有的 `review_publish` 排在导出后面 → **用户提交的评测迟迟不公开**，而这正是 worker 单元注释里点名的头号故障现象（`deploy/dsh-review-worker.service:5-6`："提交了却永远不公开"的第一嫌疑就是它没起）。
- 单任务上限 `TASK_DEADLINE_MS = 300s`（constants.ts:52），大账号导出若超 5 分钟会被 `recoverExpired` 回收 + 重试 3 次，**每次重试都白干一遍**（无断点续做），最终 failed，用户等 15 分钟拿到失败。

**③ 建议**：
- 导出任务 `schedulingGroup='user_export'`，并**在同用户维度做去重**（`deduplication_key = user_export:<userId>`），让连点复用同一任务而不是排 3 个；
- 给 claim 加**按组并发上限**（如 `user_export` 最多占 1 个槽位），或至少明确写出"接受此风险"并给出监控项；
- 超过 deadline 要考虑分片续做，或明确写出"超时即失败"的降级口径与用户可见文案。

---

## P1 — 应在同一版修掉

### 5. `me_export` 不在限流动作枚举里，这行代码编译不过

**① 问题在哪**：proposal.md:24 `await enforceRateLimit({ action: 'me_export', ctx })`。

**② 后果**：`RateLimitAction` 是闭合联合类型（`src/server/platform/rate-limit.ts:56-65`），不含 `me_export`；`RATE_LIMIT_RULES` 是 `Record<RateLimitAction, RateLimitRule>`，少一个键就编译失败。更要紧的是**运维语义**：导出是重 I/O + 重 CPU（gzip 全量数据）的操作，它的额度不该和评论共用口径，必须是一个**明显更小**的值（如 1 次/小时）。方案没给数字，等于让实现者自己拍。

**③ 建议**：补 `me_export` 及 `RATE_LIMIT_RULES.me_export = { points: 1, durationMs: 3600_000 }`，并把数字写进 proposal（这是 SRE 会去后台查/调的值，需要文档可对齐）。

### 6. 只按 7 天 TTL 回收，没有按容量兜底；单次导出的体积无上限

**① 问题在哪**：proposal.md:57-60 只有时间维度。

**② 后果**：TTL 是"最长时间"，不是"最大占用"。极端情况（批量注册的小号各导一次、或某用户帖子极多）7 天内的累计体积无上界。gzip 对 `body_md` 压缩率好，但 JSON 结构开销大，仍是 GB 级风险。

**③ 建议**：三重上限写进方案——① 单次导出字节上限（超限降级为"分批/仅元数据"并告知用户）；② 单用户同时至多 1 个保留中的产物（新导出覆盖旧的）；③ 全局配额（如 20GB）到达后按 LRU 提前删除并打 warn。

### 7. 导出文件的权限与残留风险没有交代

**① 问题在哪**：proposal.md:57 生成流程，proposal.md:11 下载路由。

**② 后果**：导出物是**该用户全量数据**（含 `body_md` 正文、收藏夹名等），落在共享卷上。方案未提：文件 mode 是否 0600、目录是否按 user 隔离权限、下载路由是否校验 `taskId 属于当前 ctx.userId`（否则换个 taskId 就能拖别人的包——这是**水平越权**，但同时也是运维事故：数据泄露要走通报流程）。另外 `remove` 用的是 `rm({force:true})`（file-store.ts:135），删除失败静默，正好配合第 2 条的"没人清"。

**③ 建议**：写入后 `chmod 0600`；下载路由强制 `WHERE task_id=? AND requested_by=?`；删除路径记录 `removed_at` 与失败计数，纳入监控。

### 8. 可见性过滤把"可携带权"做成了"只能导出公开的"

**① 问题在哪**：proposal.md:46 `if (!resolveVisibilityInTx(db, row.id).publiclyVisible) continue;`

**② 后果**：这是**合规风险而非功能瑕疵**，但会变成运维工单。GDPR Art.15/20 的访问与可携带权针对"本人数据"，与"该内容是否对公众可见"无关。用户自己的 `needs_changes` 评测、被隐藏的帖子、软删的帖子（`forum_posts.status='deleted'`，`migrations/0037_forum.sql:79`，且该迁移注释明说软删是为了保留行）都会被静默跳过 → **导出包看起来"成功"，实则缺数据**。SRE 会收到"你们导出得不全"的投诉，且无法自证。此外 `resolveVisibilityInTx` 是**逐行一次调用**（N+1），大账号下这是主要的耗时/超时来源（见第 4 条的 300s）。

**③ 建议**：本人数据一律导出（含被隐藏/软删，附 `status` 与隐藏原因），仅对**他人**数据套可见性判据。可见性过滤改为**一次 JOIN/批量判定**，不做逐行查询。

---

## P2 — 明确记录取舍即可

### 9. `assertCanWrite` 挡禁用账户 = 禁用用户导不出自己的数据

**① 问题在哪**：proposal.md:21 `assertCanWrite(ctx)`。

**② 后果**：`assertCanWrite` 对 `accountStatus !== 'active'` 抛 `disabled()`（`src/server/platform/context.ts:33-40`）。被禁用账户同样有权取回自己的数据，且这类用户往往正是最爱投诉的人。选 `assertCanWrite` 说明作者只把它当"防刷"，没想合规面。

**③ 建议**：显式记录取舍。若坚持禁用，须在方案里写明理由与替代通道（如人工申请），否则改用一个"仅登录+活跃会话"的弱校验。

### 10. 文档措辞与运维观测缺口

**① 问题在哪**：proposal.md:65"不影响其他用户"。

**② 后果**：按第 1/4/6 条，这句话目前**不成立**。另外方案没有一条"运维怎么看它是否健康"：队列积压怎么查、失败率怎么看、产物总量在哪看。当前 health 明确"不做业务统计"（health.ts:5-9），意味着导出功能上线后**没有任何现成面板**。

**③ 建议**：把预期改为有依据的说法（"单用户导出受限于 X，不占用超过 1 个并发槽位，超限拒绝"），并在方案补"可观测性"小节：导出任务 queued 数、失败率、产物总字节、最近清理时间——这四个数落进 `scripts/diagnose.mjs` 或后台页，SRE 才有抓手。

---

## 附：需要方案作者确认的开放问题

- 归档格式里的 `schema_version`（proposal.md:63）以什么为准？建议直接复用 `selfCheck` 返回的迁移版本号，避免第二个版本号来源。
- 若 userId 对应的库行已不存在（注销流程未幂等），导出任务应返回 `skipped` 还是 `error`？建议 `skipped` + 明确 errorCode，避免无意义重试 3 次。