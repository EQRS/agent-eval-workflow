# 评审：用户数据导出方案 · 性能工程师视角

> 评审对象：`atb/tasks/m13-data-export/environment/proposal.md`（65 行）
> 评审人身份：**性能工程师**（只看内存与生成时长、DB 读写与锁竞争、产物体积、清理成本、并发与配额；不评合规判定）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**按现方案实现，一次导出就能把 web 进程的同步阻塞与磁盘一起打满**。方案有 5 处缺少数量级界定：无分页/流式、无大小上限、无并发闸门、TTL 清理无归属、可见性判定被写成逐行现算。

---

## 摘要（前 5 条最重要）

1. 【严重】**§4 的 `resolveVisibilityInTx(db, row.id)` 在循环里逐行调用——每条评测要做 4 次查询，2000 条评测 = 8000 次同步 SQL。**
   依据：`proposal.md:46-51`；`src/server/modules/content/index.ts:367-369` → `resolve()` `:153-166` → `readControlRow` `:91-100`（1 次 SELECT）
   + `hasPublishedRevision` `:106-151`（**3 次** SELECT：reviews JOIN、`sqlite_master` 探测、pack_reviews JOIN）；
   其中 `sqlite_master` 那两次（`:120-122`、`:137-139`）**每次调用都重新问一遍「表在不在」**，批量场景里纯属浪费。
   后果：N 条记录 = 4N 次同步 SQL；better-sqlite3 是同步驱动（`src/server/platform/db.ts:6-8` 明写「等待会阻塞所属线程」），
   2000 条约 8000 次往返 ≈ 0.3-0.8s 纯 CPU；上万条直接进秒级，且全部压在 web 的 Node 事件循环上。
   建议：导出**不要逐行判定**——用一条 SELECT 把 `withdrawn_at / hidden_at / published_revision_id` 投影出来，复杂度 O(1)。

2. 【严重】**§1「异步后台任务」并没有真的异步：任务由 worker 执行，而 worker 是单进程串行的。**
   依据：`proposal.md:15`；`src/worker/main.ts:190-205`（一轮 claim 最多 4 个、**串行 `await execute`**，`:199` 注释写明「本切片按顺序执行」）、
   `src/server/modules/tasks/constants.ts:16,45`（`MAX_CONCURRENT_TASKS=4`、`TASK_DEADLINE_MS=300s`）。
   后果：一个大导出会占满 worker 的 5 分钟截止，期间所有评测发布、评论发布任务排队；超时后 `recoverExpired`（`service.ts:402-440`）
   把它当失败回收 → **下一轮从头重写整个文件**；连续 3 次超时任务直接 `failed`，用户永远拿不到文件。
   建议：给导出任务单独的截止（如 1800s），或每 500 行检查剩余时间并干净收尾；把「已写行数」写进 `progress` 以便判断是否可续。

3. 【严重】**§3+§4 把导出写成「逐行点查 + 内存累积」，没有任何流式/分页。**
   依据：`proposal.md:24-51`（白名单列，但没有 LIMIT/OFFSET/游标，也没有「边写边落盘」）；
   对比本仓既有范式：`scripts/backup.mjs:63-65` 用 SQLite 在线热备 `db.backup()`（**不阻塞写入**）、`:78-94` 逐表计数对账，而不是把全库读进内存。
   后果：5000 帖 + 2000 评测，`body_md` 单条几 KB → 数十 MB 的 JS 字符串驻留内存，`JSON.stringify` 中间结果再膨胀 2-3 倍；
   且与全站请求共用同一进程（`src/server/platform/db.ts:67-74` 进程级单例）。
   建议：**游标分页 + 边写边压**：每批 500 行（`WHERE id > ? ORDER BY id LIMIT 500`）直接写进 gzip 流，单条序列化完立刻丢引用；
   峰值内存 = 500 行 + 一个 64KB 压缩缓冲，与用户数据量无关。

4. 【高】§5 的产物落在 `data/exports/<userId>/<taskId>.json.gz`，**与 SQLite 主库同盘**，且无大小上限、无清理归属。
   依据：`proposal.md:53-57`；`docker/compose.yaml:41,65,92`（`.data` 卷同时是数据库与媒体根）；
   `deploy/dsh-review-backup.service:26` 与 `scripts/backup.mjs:99-108`（备份会把整个媒体目录 `tar -czf`）；
   清理口径的既有纪律：`src/server/platform/rate-limit.ts:182-184`「**不开启库的定时清理：它会 setInterval，而我们要进程能被干净关掉**」。
   后果：① 用户数 × 导出次数 × 体积，7 天后才清，可把数据盘写满；② 盘满时 SQLite 写直接 `SQLITE_FULL`，**全站写操作瘫痪**（`db.ts:44-63` 无容量保护）；
   ③ 导出目录若在媒体根下，每次备份都会连带打包，备份体积与耗时同步膨胀；④「定时清理」与 `rate-limit.ts:182-184` 的进程纪律冲突。
   建议：产物放独立卷并在备份中排除；单次导出硬上限（如 200MB / 50 万行）；清理挂在已有的一次性任务上（worker 每轮开头或 oneshot service），**不要 setInterval**。

5. 【高】§2 的频控写法与仓库 API 不匹配（编译不过），且缺 `RATE_LIMIT_RULES` 条目。
   依据：`proposal.md:18-22` 写的是 `enforceRateLimit({ action: 'me_export', ctx })`——实际签名是
   `enforceRateLimit(db, action, userId, points?, override?)`（`src/server/platform/rate-limit.ts:196-208`），**位置参数、第一个是 db**；
   `RateLimitAction` 联合类型（`rate-limit.ts:46-55`）里**没有** `me_export`，规则表（`:68-134`）也需新增条目，否则 `limiterFor` 会拿 `undefined` 的 `points` 构造 limiter。
   建议：`RATE_LIMIT_RULES` 增 `me_export: { points: 1, durationMs: 86400000, description: '每用户每天最多导出 1 次' }`；
   调用写成 `const ticket = await enforceRateLimit(db, 'me_export', ctx.userId, 1, await rateLimitOverrideFor(db, RATE_LIMIT_RULES.me_export))`，
   失败路径 `refundQuietly(ticket)`（范式见 `src/app/api/me/roles/applications/route.ts:41-56`）。**1 次/天/用户**同时是并发闸门的廉价替代（见 P-04）。

---

## 完整问题清单（按严重度）

### P-01 【严重】逐行调用 `resolveVisibilityInTx` = 4N 次同步 SQL
- **现场**：`proposal.md:46-51`。
- **依据**：`src/server/modules/content/index.ts:153-166`（`resolve` 内部两次调用）、`:91-100`（1 次 SELECT）、`:106-151`（3 次 SELECT，含 2 次 `sqlite_master` 探测）；`src/server/platform/db.ts:6-8`。
- **后果**：见摘要 1。即使单次 `prepare().get()` 只要几十微秒，8000 次也是 0.3-0.8s；行数上万直接进秒级，且**串在 web 的事件循环里**。
- **建议**：把可见性降维成一条带 `EXISTS` 的 SELECT 列投影（`WHERE ce.author_id = ? AND r.id > ? ORDER BY r.id LIMIT 500`），判定在内存里做，口径与 `content/index.ts:158-165` 一致。

### P-02 【严重】「异步」实际是 worker 串行执行 + 300s 截止
- **现场**：`proposal.md:15`。
- **依据**：`src/worker/main.ts:190-205`（串行）、`src/server/modules/tasks/constants.ts:16,45`、`src/server/modules/tasks/service.ts:402-440`（超时回收重排）。
- **后果**：见摘要 2。补充：若把导出内容塞进 `progress`，会把这个全站最热的表撑大（见 P-06）。
- **建议**：文件写盘，`progress` 只存 `{ rowsWritten, bytesWritten, cursor }`；导出任务单独截止或做时间预算检查。

### P-03 【严重】无分页/流式/大小上限，内存与产物体积双双失控
- **现场**：`proposal.md:24-51`、`:53-57`。
- **依据**：`scripts/backup.mjs:63-65,78-94`（在线热备、逐表对账的既有范式）；`db.ts:53`（WAL）与 `db.ts:56`（`busy_timeout` 5000ms）。
- **后果**：一次导出若在一个 `withRead` 里读完全部行，等于**一个长读事务** → WAL 无法 checkpoint、WAL 文件持续增长，checkpoint 时与写入抢锁，web 侧写请求开始 `SQLITE_BUSY`。
- **建议**：每批 500 行各开一个小读事务（不重叠），保证 WAL 可回收；行数与字节双阈值，开始前用一次 `count(*)` 预判；压缩用 `zlib` 流式接口，不要先 `JSON.stringify` 整个数组。

### P-04 【高】无并发闸门：同一用户可叠加多个导出任务
- **现场**：`proposal.md:10-13`。
- **依据**：任务去重键是 `(task_type, deduplication_key)`（`0007_task_type_plugin_discovery.sql:47`），键里必须带 taskId 才能让每次导出独立 → **没有天然去重**；`claimWithin` 一次最多 4 个并发（`main.ts:191-195`、`constants.ts:16`）。
- **后果**：连点 4 次 = 4 个任务同时跑 = 4 倍 DB 读 + 4 份磁盘产物；更严重的是 **worker 的 4 个并发位被同一个人占满**，全站内容发布停摆。
- **建议**：频控 1 次/天/用户 + 入队前查「是否已有 `me_export` 处于 queued/running」；写盘前判剩余空间，ENOSPC 时干净收尾别留半个 gzip。

### P-05 【高】产物与主库同盘 + 清理无归属，盘满会瘫痪全站写
- **现场**：`proposal.md:55-57`。
- **依据**：`docker/compose.yaml:41,65,92`；`deploy/dsh-review-backup.service:26`、`scripts/backup.mjs:99-108`；`src/server/platform/rate-limit.ts:182-184`。
- **后果**：见摘要 4。补充：cleanup 若写成 `setInterval`，会与 worker 的 SIGTERM 优雅退出（`main.ts:248-255`）打架——定时器持有 handle，进程无法按时退出，`docker stop_grace_period: 30s` 到期被 SIGKILL，留下半个文件。
- **建议**：清理放进每轮 tick 开头的一次性批处理（`main.ts:170-206` 已有 `recoverExpired` 的位置），每次最多删 50 个，**没有后台定时器**。

### P-06 【中】导出进度若落进 `background_tasks.progress`，会把全站最热的表撑大
- **现场**：方案没说明进度/结果存在哪里。
- **依据**：`claimWithin` 的选取 SELECT 每次都取 `payload`（`service.ts:122-131`），`finish` 会重写整行（`:344-379`）。
- **后果**：progress 一旦是 MB 级，每次 claim/finish 都搬运这几 MB；claim 是每 10 秒一次的路径（`constants.ts:25,48`）→ 每 10 秒搬一次大对象。
- **建议**：progress 只存摘要，明细写文件——本仓已有先例（发现任务的 summary，`src/worker/handlers.ts:352`）。

### P-07 【中】§4「与页面同一套判据」对帖子根本跑不通（这是代码事实，不只是语义之争）
- **现场**：`proposal.md:46-51`。
- **依据**：`resolveVisibilityInTx` 只认 `content_entities` 的撤/隐状态（`content/index.ts:153-166`），而帖子是 `forum_posts.status` 的另一套软删（`0037_forum.sql`）；`readControlRow` 查不到行时直接 `throw err.unavailable`（`content/index.ts:98`）。
- **后果**：导出帖子会抛 `TARGET_UNAVAILABLE` → 任务进 `error` → 重试 3 次 → `failed`；性能上表现为每次跑满 3 次重试、写 3 次半成品文件。
- **建议**：可见性按表分别定义（评测看 `content_entities`，帖子看 `forum_posts.status`），并在导出前一次性投影出来。

### P-08 【低】无导出成本的可观测（谁导了多大、花了多久）
- **现场**：`proposal.md:62-65`。
- **依据**：可观测手段是任务 `progress` 与 `operation_records`（`0001_baseline.sql:411-431`，`0042` 已禁 UPDATE/DELETE）。
- **后果**：磁盘被谁占满、哪个用户导出了 3 次，事后只能翻 journal。
- **建议**：导出完成写一条 `operation_records`（`action_type='me.export'`，`changes` 放 `{ rows, bytes, durationMs }`），与 `plugin.discovery.run` 同形（`src/worker/handlers.ts:319-345`）。

---

## 结论与建议

**方案在性能维度不可按原样落地**：它把导出写成「逐行现算可见性 + 全部读进内存 + 落盘到主库同盘 + 靠未定义的定时器清理」，
而这四条恰好命中本仓三个硬约束：**better-sqlite3 同步阻塞**（`db.ts:6-8`）、**单文件 SQLite + WAL**、**worker 串行 + 300s 截止**。

**最低改造（不改变方案结构，只补数量级）**：
1. 分页 + 流式：每批 500 行、每批一个小读事务、直接写 gzip 流（P-03）。
2. 可见性一次投影：一条带 `EXISTS` 的 SELECT 取代 4N 次逐行调用；帖子单独口径（P-01、P-07）。
3. 配额 + 并发闸门：`RATE_LIMIT_RULES` 增 `me_export`（1 次/天/用户）+ 入队前查「已有在跑」（P-04、摘要 5）。
4. 大小上限双阈值（行数 + 字节），超限在开始前用 `count(*)` 拒绝（P-03）。
5. 产物离开主库卷、备份排除、清理走一次性批处理（不 setInterval）（P-05）。
6. 进度只存摘要（P-06）；完成写一条 `operation_records`（P-08）。

改完之后的量级：单次导出 = O(行数/500) 个小事务 + 一次流式压缩，**web 进程阻塞时间接近 0，磁盘占用有上限，清理可预期**。

> 评审人声明：本报告只覆盖性能/容量/并发/IO 视角。「删掉不可见内容是否违反数据可携带权」、导出范围是否足够、产物加密与访问控制属于合规/安全视角，
> 本文不代替它们的结论。但请注意 P-07 是代码事实：`resolveVisibilityInTx` 对帖子会直接抛错，不论合规怎么定，按现方案都跑不通。
