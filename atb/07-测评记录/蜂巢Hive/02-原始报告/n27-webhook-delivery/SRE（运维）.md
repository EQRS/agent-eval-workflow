# n27-webhook-delivery · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_题库全测_验证单元_0015
> 模型 deepseek-chat · 7655 字 · 41 次工具调用

---

## SRE 评审：tasks/n27-webhook-delivery/environment/proposal.md

### 摘要（先说结论）

**不建议按现方案实施。** 方案只有 37 行，把「异步 POST + 重试 3 次」当成 Webhook 的全部，但它要接入的是一套**已有明确任务队列、审计、健康检查与单写者 SQLite** 的生产系统。方案缺的恰好是运维必需的四件事：**谁执行投递**、**状态存在哪**、**失败怎么观测**、**如何限制影响范围**。逐条如下。

---

### 1. 【阻断】没有「谁执行投递」的落点，`await fetch` 在本仓无处可放

① **问题**：方案 §2 只写「异步 POST 到订阅的 URL」，没有说明投递进程/调用点。插件更新的写入路径在 Web 侧（`src/server/workflows/publish-review.ts`、`publish-comment.ts`）和 worker 侧（`src/worker/handlers.ts:createHandlers`）都存在。

② **后果**：无论放哪边都违规：
- 放 Web 请求路径 → 违反 `src/server/platform/db.ts:6-8` 明文纪律「不在事务内做网络请求」，且 `src/server/platform/tx.ts:33-42` 的 `withTransaction` 回调必须同步，`await fetch` 根本进不去；硬塞到事务外则是请求线程持有一个外部 HTTP 长尾，上游订阅者慢就等于本站页面慢。
- 放 worker → 现队列 `src/server/modules/tasks/service.ts` 的 `task_type` 是**库内 CHECK 闭集**，只有 `review_check/review_publish/comment_check/comment_publish/plugin_discovery`（`migrations/0007_task_type_plugin_discovery.sql:26-28`）。新增一种任务类型必须再走一次**重建 background_tasks 表**的迁移，而该迁移自带 `-- dsh:migration-transaction=off`（`0007:1,11-18`），失败会留半成品。方案对此零字。

③ **建议**：明确「Webhook 投递 = worker 的一个任务类型」，并**在方案里就写出**需要 `0043` 迁移扩展 `task_type` CHECK（沿用 `0007` 的重建手法 + 索引重建），同时写清迁移失败的回滚口径。若不愿动 `task_type`，唯一可接受替代是复用 `plugin_discovery` 之类的既有类型——但这会污染 progress 语义，不建议。

---

### 2. 【阻断】订阅表缺「投递状态」，重试在库里无锚点

① **问题**：§1 的 `webhook_subscriptions` 只有 `id/user_id/url/secret/created_at`，§3 的「失败重试 3 次」没有任何持久化字段；§2 的代码里 `fetch` 的返回值被**完全丢弃**（没有 `res.ok` 判断，没有 `response` 变量）。

② **后果**：
- 按字面实现，「异步」= 内存里 `await` 一次，进程重启/worker OOM 后**投递丢失且无记录**。这与本仓 `src/server/platform/heartbeat.ts:4-8` 明确要防的「静默死亡」是同一类问题，方案自己制造了一个新的静默失败面。
- 重试 3 次无退避无间隔（对比 `src/server/modules/tasks/constants.ts:38-42` 已有的 `RETRY_BACKOFF_MS = [30s, 120s]`），订阅方连续 3 次被同一时刻打爆。
- `fetch` 默认**无超时**。订阅方 hang 住 → 投递线程/请求挂死，`TASK_DEADLINE_MS = 300s`（`constants.ts:47`）到点才被 `recoverExpired` 回收，期间占着 `MAX_CONCURRENT_TASKS = 4` 的一个名额。

③ **建议**：方案必须落三件事——(a) 投递记录落库（新建 `webhook_deliveries`：`subscription_id / event_key / state / attempt_count / next_attempt_at / last_status_code / last_error_code`，并对 `(subscription_id, event_key)` 加 UNIQUE 做幂等，参照 `migrations/0008_notifications.sql` 的 `deduplication_key` 手法）；(b) 复用队列的退避常量与「最多 3 次 → 终态 failed」语义，不要自创一套；(c) `fetch` 必须带 `AbortSignal.timeout()`（建议 10s 量级）与 `res.ok` 判断。

---

### 3. 【严重】`secret` 是明文存储，且签名机制根本没写

① **问题**：§1 `secret TEXT NOT NULL` 无任何脱敏/加密说明；§2 的 headers 只有 `content-type`，**没有签名头**。

② **后果**：
- 库里明文存第三方密钥，与 `src/server/platform/log.ts:26` 的敏感键黑名单纪律（`token|secret|password|...` 一律 redact）方向相反——日志在防泄漏，表里却在明文存。备份产物（`docker/compose.yaml` 的 `BACKUP_ROOT: /app/backups`）会把这些 secret 一起带走，而备份保留 `BACKUP_KEEP=14`。
- 无签名 = 订阅方无法验证来源，本仓「审计不可变」的努力（`migrations/0042_audit_immutable.sql:17-27`）在对外通道上是敞开的：任何人拿到 URL 就能伪造事件。方案 §2 直接 `JSON.stringify(event)` 且事件体内容未定义，连最小防重放（时间戳+nonce）都没有。

③ **建议**：签名头（`X-DSH-Signature: t=<ts>,v1=<HMAC-SHA256(ts.body)>`）写进方案，且**校验方必须有重放窗口**；`secret` 改为只存摘要不可行（要拿原文算 HMAC），因此至少要：库里存 secret 但明确「备份产物按凭据对待」、日志/诊断脚本永不打印（`scripts/diagnose.mjs:56-72` 的 `envState` 是「只看配没配」的正确范式，新字段要沿用）、并提供轮换路径（`secret_rotated_at` + 双 secret 并存窗口）。

---

### 4. 【严重】`REFERENCES users(id)` 默认 RESTRICT，与全站删号/禁用流程冲突

① **问题**：`user_id TEXT NOT NULL REFERENCES users(id)` 未写 `ON DELETE`，SQLite 默认 `NO ACTION`（等价 RESTRICT）。

② **后果**：本仓所有业务外键都显式写了 `ON UPDATE RESTRICT ON DELETE RESTRICT` 且**刻意**如此（见 `migrations/0001_baseline.sql` 各表、`0008_notifications.sql:33-35` 注释「用户/评论/内容实体被引用着就删不掉，避免通知指向空气」）。Webhook 订阅是**用户自建的可丢弃配置**，把它也加成 RESTRICT，等于给「禁用/清理账号」流程永久加钉子：一个用户订了一个 webhook，这个账号就再也清理不掉，且清理脚本会在半路 ABORT。相比之下 `migrations/0034_featured_subscriptions.sql:17` 的订阅表用的是 `ON DELETE CASCADE`——同类语义已经有先例，方案却选了默认值。

③ **建议**：改为 `ON DELETE CASCADE`（订阅是配置，不是事实），并在方案写明「用户删除 → 订阅与未投递任务一并消失」的语义；同时补 `CREATE INDEX idx_webhook_subscriptions_user ON webhook_subscriptions(user_id)`，否则「某用户的订阅列表」会全表扫（`0008` 与 `0034` 都建了对应索引，方案一个索引都没有）。

---

### 5. 【严重】没有出口信任边界，SSRF 面直接对外

① **问题**：`url TEXT NOT NULL` 无任何 CHECK，§2 直接对用户提供的 URL 发起 `fetch`，且没有出站网络策略说明。

② **后果**：`docker/compose.yaml` 里 web/worker/nginx 同处一个 `dsh-plugin-review` 网络，worker 能直连 `web:3000`、也能打到宿主机与内网。用户填 `http://web:3000/api/...` 或 `http://169.254.169.254/...`，本站自己就是被探测的内网资产。方案对此完全没有防护。

③ **建议**：方案增加出站约束并写进实施项：(a) `url` 写入前校验 scheme 仅 `https`、禁止 IP 字面量与 `localhost`；(b) **投递时**再解析一次并对内网段（RFC1918 / 169.254 / ::1 / 容器网段）拒投（DNS rebinding 只能靠投递时校验兜）；(c) 明确 worker 的出站是否要经带策略的 egress（若短期做不到，至少在方案里登记为已知风险，并给「关闭 webhook 功能」的一行 env 开关——参照 `src/worker/scheduled.ts:33-45` 的 `DSH_REVIEW_DISCOVERY_DAILY=0` 硬刹车范式）。

---

### 6. 【高】缺少运维必需的观测口径：我拿什么判断「webhook 在正常工作」

① **问题**：方案「预期」只有两句「能及时收到事件 / 失败可重试」，没有任何可观测项。

② **后果**：本仓的运维诊断链路是明确的 `src/server/platform/health.ts`（`buildHealthReport` + `?worker=1` 让 `/api/health` 返回 503）→ `scripts/diagnose.mjs` → `src/worker/main.ts` 的 `startHeartbeat` 心跳（`WORKER_STALE_MS=30s / DEAD_MS=120s`，`heartbeat.ts:22-28`）。webhook 若按方案实现，**这三处都看不到它**：worker 心跳照常（哪怕所有投递都 404），health 照常 200，diagnose 的 `problems` 不会包含「积压了 500 条待投递」。等于新增了一个只能靠人肉翻日志（`log.ts` 是 stdout json-line）发现的功能。

③ **建议**：方案补三件：(a) 在 `diagnose.mjs` 增加「待投递条数 / 最老一条滞留时长 / 最近 24h 失败数」并纳入 `problems` 判定；(b) 失败终态用 `log.error` 打**受限错误码**（沿用 `handlers.ts` 的 `terminal(errorCode, summary)` 范式，绝不打印 URL 全文与响应体）；(c) 定义「投递积压超过 N 分钟」是否要影响 `/api/health`——建议**不影响**（与 `requireWorker` 默认不传的取舍一致：别让第三方的问题重启我们的容器），但必须在方案里显式写这个取舍。

---

### 7. 【高】SQLite 单写者 + `synchronous=FULL` 下，「异步投递」的写入成本未评估

① **问题**：方案对投递的库写入零描述。

② **后果**：`db.ts:52-56` 是 WAL + `synchronous = FULL`，每个事务要 fsync。若按本文建议落 `webhook_deliveries`（第 2 条），每次投递成功/失败都是一次写事务，与任务队列的 `claim/renew/finish`（`service.ts`）争同一个写者。`MAX_CONCURRENT_TASKS=4` 个并发投递 + 4 个业务任务时，写入排队会让 `busy_timeout=5000` 成为真实风险。方案没给任何量级估算（订阅数、事件量、单事件大小）。

③ **建议**：方案必须写清容量假设（多少订阅者 × 每日多少事件），并把投递状态更新**合并批写**（一轮任务里 n 条投递结果一次事务写完），而不是每条一次 `UPDATE`。另外明确 `webhook_deliveries` 的**保留策略**（谁清、留多久、按什么删）——本仓已有 `rate_limiter_flexible` 的过期清理先例（`0006_rate_limiter.sql:31-32` 的 `idx_rate_limiter_expire`），照这个范式给投递记录建 `(state, next_attempt_at)` 或 `created_at` 索引，否则这张表会单向增长。

---

### 8. 【中】迁移脚本要遵守本仓既有约定，方案未提

① **问题**：方案把建表 SQL 当作「等等」的示意，没有当作**迁移文件**对待。

② **后果**：本仓纪律是「权威结构是 `migrations/*.sql`」（`src/server/platform/migrate.ts:1-13`），且 `selfCheck` + `REQUIRED_TABLES` + `scripts/diagnose.mjs:30-38` 三处清单需要同步；`0009_review_verdict.sql` / `0029_...` 都是小步 `ALTER`，纯增量迁移**不需要** `dsh:migration-transaction=off`（`0042_audit_immutable.sql:14-15` 说得很清楚）。方案若照第 1 条去动 `task_type`，就必须带 `off` 标记并自管 BEGIN/COMMIT——这个差别必须写进方案，否则实施者会二选一蒙。

③ **建议**：方案里直接给出迁移文件编号（下一个是 `0043`）、是否需要 `dsh:migration-transaction=off`、以及需要同步修改的三处清单（`migrate.ts` 的 `REQUIRED_TABLES`、`diagnose.mjs:30-38`、`0007` 式索引重建说明）。

---

### 9. 【中】「重试 3 次」的成功判据缺失（幂等 / 订阅方语义）

① **问题**：§3 只有「失败重试 3 次」，没定义「失败」是网络异常、非 2xx、还是含 429/5xx；也没定义 4xx（订阅方已删该 endpoint）是否该重试。

② **后果**：若 4xx 也重试，等于对已废弃的 endpoint 打三次无效请求；若 2xx 之外的 3xx 被当成成功，`fetch` 默认会跟随重定向，最终可能投递到用户没授权的域（与第 5 条 SSRF 叠加）。多次重试还可能让订阅方**重复收到同一事件**——本仓已经有成熟的幂等范式（`0008_notifications.sql:26,40` 的 `UNIQUE(deduplication_key)`、`handlers.ts` 的 `already_published` → success），Webhook 事件必须同样带稳定的 `event_id`。

③ **建议**：明确判据矩阵：`2xx=成功`；`410/404=终态失败不重试`；`429/5xx/网络错误=可重试`；`3xx=不跟随（`redirect: 'manual'`）按失败处理`。事件体必须含 `event_id` 与 `occurred_at`，并在方案里承诺「同一 `event_id` 可重复投递，订阅方按 `event_id` 去重」。

---

### 结论

按现方案字面实施，会把一个**无执行落点、无状态、无签名、无超时、无观测、无出口约束**的 `fetch` 塞进一套纪律很严的系统。放行前必须在方案里补齐：

1. 投递执行者 = worker 任务类型（含 `task_type` CHECK 扩展迁移及其 `migration-transaction=off` 口径）；
2. `webhook_deliveries` 表（幂等键 + 重试状态 + 保留策略）；
3. HMAC 签名 + secret 的日志/备份/轮换口径；
4. `ON DELETE CASCADE` + `user_id` 索引；
5. 出站 URL 校验与内网拒投 + 一行 env 硬刹车；
6. 超时（`AbortSignal.timeout`）+ 退避复用 `RETRY_BACKOFF_MS`；
7. `diagnose.mjs` / `/api/health` 的观测口径与「是否影响 200/503」的显式取舍；
8. 容量假设与写入批合并。

第 1–3 条不落字，SRE 侧应该阻止推进——它们不是实现细节，而是「这个功能上线后，出事时我们能不能定位、能不能关掉」的前提。