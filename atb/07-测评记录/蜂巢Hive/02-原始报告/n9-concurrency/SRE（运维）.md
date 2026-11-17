# n9-concurrency · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0004
> 模型 deepseek-chat · 7632 字 · 32 次工具调用

---

我已读完全部相关材料。以下是 SRE（运维）视角的评审。

---

# SRE 评审：收藏夹条目排序并发更新方案

评审对象：`atb/tasks/n9-concurrency/environment/proposal.md`（共 34 行）
核实依据：`migrations/0012_interactions.sql`、`0019_favorite_folders.sql`、`0039_favorite_folders_target_kind.sql`、`0042_audit_immutable.sql`、`src/server/platform/{db,tx,migrate,health,log,rate-limit}.ts`、`src/server/modules/interactions/index.ts`、`src/instrumentation-node.ts`

**总体判断：不建议按现方案上线。** 它的核心并发手段（无上限 while 重试）恰好会触发本仓已经写进代码注释的两个物理限制——better-sqlite3 是同步驱动、等待会阻塞线程（`src/server/platform/db.ts:8-9`），以及 `busy_timeout` 只有 5000ms（`db.ts:21`、`db.ts:59`）。下面按严重度排。

---

## P0-1 重试循环是**无上限死循环**，且会把 Node 主线程钉死

- **哪部分有问题**：`proposal.md:11-24` 的 `while (true)`。循环里没有次数上限、没有退避、没有 deadline，且 `break` 的唯一条件是 `result.changes === 1`。
- **后果**：
  - `db.ts:8-9` 明写「better-sqlite3 是同步驱动，等待会阻塞所属线程」。这个 `while` 是**同步**的：每次 `prepare().run()` 一旦撞上写锁，会在 `busyTimeoutMs=5000`（`db.ts:21`）内**同步阻塞整个事件循环**。单个请求最坏 5s 内进程不响应任何一个请求；N 个用户同时拖拽 = N×5s 串行排队，整个 Web 进程假死。
  - 高争用下 CAS 永远失败（每次重读都能读到新 version），循环不会退出 → 该请求永不返回，客户端超时后**前端重发**（`proposal.md:4` 说「每次拖拽都发一个请求」）→ 放大争用，正反馈雪崩。
  - 运维可见的现场是：`/api/health`（`health.ts`）探活也走同一进程的事件循环，会一起超时 → 容器健康检查失败 → 编排器重启 → WAL 恢复 + 重启风暴。这类故障在日志里**一个字都没有**，正好复现 `heartbeat.ts:4-5` 描述的那种「静默死亡」形态。
- **建议**：
  1. 循环加硬上限（如 3 次）+ 指数退避 + 每轮检查 deadline（总预算 ≤ 200ms）；
  2. 超限后**抛业务错误**（复用 `errors-http.ts:36` 的 `VERSION_CONFLICT → 409`），让前端重新拉取真序再写，而不是在服务端硬扛；
  3. 把整段放进 `withTransaction`（见 P0-2），并且**绝不**在事务里做等待。

---

## P0-2 SELECT 与 UPDATE 分处两条独立语句、无事务包裹 → 冲突窗口被人为放大

- **哪部分有问题**：`proposal.md:12-21`。`SELECT position, version` 和随后的 `UPDATE ... AND version = ?` 是两个独立语句，提案中没有 `BEGIN`。
- **后果**：本仓所有写路径都要求走 `withTransaction`（`tx.ts:1-12`「跨模块流程创建一次短事务…不得各自提交独立事务」）。而 `interactions/index.ts` 里现存的收藏写入（如 `favoriteToDefaultFolder` 的 `ON CONFLICT ... DO UPDATE`、`ensureDefaultFolderInTx`）全部在 `withTransaction` 内。新方案把「读-判-写」拆成两条自动提交语句，等于在**两条语句之间主动开了一个可被任意插入的窗口**——这正是它需要 while 重试的原因，属于自己制造问题再自己重试。且它与现有事务路径混跑时，重试次数完全不可预测。
- **建议**：用 `db.transaction()`（`better-sqlite3` 的 `IMMEDIATE` 事务）把 SELECT+UPDATE 包起来。SQLite 的写锁一旦在事务开头拿到，后续 CAS 就几乎不会失败，重试从「常态」降为「异常」。并且明确 `BEGIN IMMEDIATE`，避免读事务升级写事务时的 `SQLITE_BUSY`（那种 busing **不受 busy_timeout 保护**，会直接抛错）。

---

## P0-3 方案读的表与字段，**在当前库结构里不存在**

- **哪部分有问题**：`proposal.md:9-10` 的 `ALTER TABLE plugin_favorites ADD COLUMN position/version`，以及 `proposal.md:13` 的 `WHERE user_id = ? AND plugin_id = ?` 单行语义。
- **后果（带行号）**：
  - `plugin_favorites` 建表在 `0012_interactions.sql:33-38`，主键是 `(user_id, plugin_id)`，**没有 position / version**。提案声称要加这两列——但**没有给迁移文件名**，而本仓迁移是**严格按文件名顺序执行**的（`migrate.ts` 的 `listMigrationFiles` + `localeCompare`），当前目录已到 `0042_audit_immutable.sql`。补加列必须是一个 ≥ `0043` 的新迁移文件，提案里完全没有这一节。
  - 更关键：`0039_favorite_folders_target_kind.sql:1` 用 `dsh:migration-transaction=off` **重建**过 `favorite_folders`，并引入 `folder_id`（`0019:26-28`）。收藏现在是**分夹**的，「一个用户的一个插件只有一行」（`0019:8-9`）——所以 `position` 必须在 `(folder_id)` 范围内唯一/有序，而不是按 `(user_id, plugin_id)` 单行排序。提案的 SQL 语义与 `0039` 之后的模型不匹配：`SELECT ... WHERE user_id=? AND plugin_id=?` 只能取到一行，**根本无法表达「批量重排」**。
  - `0019:29-30` 建的索引是 `idx_plugin_favorites_by_folder ON plugin_favorites(folder_id, created_at DESC, plugin_id)`。列表读路径（`interactions/index.ts:320-321` 的 `ORDER BY f.created_at DESC, f.post_id ASC`、插件侧 `readFavoritePlugins`）都按 `created_at` 排。加 `position` 后这些 ORDER BY 必须改，**提案一字未提** → 上线后「排序改了但不生效」。
- **建议**：提案必须补齐「迁移 0043」小节：列定义、`folder_id` 维度上的 `UNIQUE(folder_id, position)` 或 `CHECK(position >= 0)`、以及**替换现有排序索引**。参考 `0042_audit_immutable.sql` 的写法——纯增量、明确声明「不需要 transaction=off」。

---

## P0-4 并发正确性论证缺前提：`busy_timeout` 与 `SQLITE_BUSY` 会绕过 CAS

- **哪部分有问题**：`proposal.md:26-30` 的「预期：顺序不会错乱 / 并发高时自动重试 / 用户感知不到冲突」。
- **后果**：三条预期**全部没有失败路径的描述**。具体：
  - CAS 只能解决「版本被改了」，解决不了「拿不到写锁」。`db.ts:59` 的 `busy_timeout=5000` 到点后 better-sqlite3 抛 `SQLITE_BUSY`，这是**异常**，不是 `changes===0`，提案的重试循环不 catch 它 → 直接 500。
  - 提案说「用户感知不到冲突」（`proposal.md:30`），但没有任何一条链路把冲突转成用户可理解的态。反过来，若按 409 返回，前端必须能自愈——提案也没写。
- **建议**：在提案里明确写清三档降级：① `changes===0` → 有界重试；② `SQLITE_BUSY` → 转 409 且**明确不打 error 日志**（否则会像 `errors-http.ts` 里 `domain_error` 那条注释说的，正常流程刷爆错误日志）；③ 重试耗尽 → 409 + 前端全量刷新。这三档要能在日志里区分。

---

## P1-1 没有限流/合并，拖拽是**高频写**动作

- **哪部分有问题**：`proposal.md:4`（「前端每次拖拽都发一个请求」）+ 全文没有任何频控设计。
- **后果**：`rate-limit.ts` 里的 `RateLimitAction`（`rate-limit.ts:57-66`）**没有 favorites/拖拽这一档**——`comment_change`、`report`、`feedback` 等都有额度，收藏排序没有。用户一次拖拽手势（drag 事件）可能产生几十次写，全部落到单文件 SQLite 的写锁上，与评论、举报、worker 的任务写入抢同一把锁（`worker/main.ts` 的 claim/finish 也是写）。这是把「UI 交互频率」直接当「DB 写频率」用。
- **建议**：
  1. **首选：前端合并**——拖拽结束（drop）时发 1 次批量请求，传整段新顺序，服务端一个事务内全量 update。这从根上消除并发内耗，比任何 CAS 都便宜。提案的「批量更新」在背景里提了一句，却给出了单条重试的实现，前后矛盾。
  2. 服务端加一档 `RateLimitAction`（如 `favorite_reorder`）并在 `RATE_LIMIT_RULES` 登记，键按 `userId`。

---

## P1-2 缺少可运维信号：没有指标、没有结构化日志、没有诊断入口

- **哪部分有问题**：整个 `proposal.md` 34 行，零个可观测性设计。
- **后果**：一旦上线，运维**无法回答**：重试发生了多少次？平均几轮？谁在死循环？`health.ts` 只报库/外键/worker 心跳/磁盘，`health.ts:1-10` 明确说「不做业务统计」，所以这个新写入热点**不会出现在任何健康项里**。日志方面，`log.ts` 提供结构化 stdout，但提案没要求打任何一条。等于把一个已知的争用点部署成黑盒。
- **建议**：
  - CAS 重试次数 > 1 时打一条 `log.warn('favorite_reorder_retry', { request_id, user_id, attempts })`；耗尽时 `log.error`。
  - 在 `health.ts` 的 checks 里加一项轻量指标（如「最近 1 分钟内 reorder 冲突数」），或至少让 `scripts/diagnose.mjs` 能查——注意保持 `health.ts:5-7` 的纪律：不要把业务数字放进公网可达端点。
  - 明确 `db.ts:8-9` 的「事务短小」红线：reorder 事务不得包含任何 Markdown 解析或外部调用。

---

## P1-3 `position` 的约束缺失会引入**不可维护的数据态**

- **哪部分有问题**：`proposal.md:9` `position INTEGER NOT NULL DEFAULT 0`。
- **后果**：没有 `UNIQUE(folder_id, position)`、没有 `CHECK(position >= 0)`、没有「重排后是否归一化 0..N-1」的约定。并发重试穿插之后完全可能出现两行同 position（都写成功、不同事务）→ 列表顺序变成**依赖 SQLite 物理行序的非确定结果**。这类问题在故障复盘时最难查：它不报错、不写日志，只是「有时候顺序怪怪的」。本仓的惯例是「约束写进迁移、用 CHECK 与部分唯一索引钉死」（`0019:16-22`、`0039:44-56` 都是这个风格），提案与之不符。
- **建议**：迁移里加 `UNIQUE(folder_id, position)`（冲突即失败，配合 P0-2 的事务可保证不会脏写）+ `CHECK(position >= 0)`；并提供一条**幂等修复语句**（`window function` 重排 0..N-1）写进迁移或 `scripts/`，供事故后一键纠正。

---

## P2-1 回滚与部署顺序未定义

- **哪部分有问题**：提案没有「迁移如何回滚」与「代码与迁移的发布顺序」。
- **后果**：`migrate.ts:70-79` 说明迁移是**按文件独立事务、失败回滚该文件、已成功不重跑**。加列的迁移本身可回滚，但**一旦有数据写进 `position`，回滚就意味着丢排序**。回滚路径存在时，`0019`/`0039` 的历史教训是：重建表类迁移需要 `transaction=off` 且要人工验证（`0039:5-13` 那段注释就是被实测钉出来的）。运维需要提前知道这次是「加列」（安全，可前滚）还是别的。
- **建议**：
  1. 明确本次是**纯加列 + 加索引**（不做表重建），因此**不需要** `dsh:migration-transaction=off`——像 `0042:15-17` 那样把这句话写进迁移头部；
  2. 明确发布顺序：**先迁移后发代码**（旧代码不认新列但 `NOT NULL DEFAULT 0` 兼容）；回滚代码时保留列，不删。

---

## P2-2 「用户感知不到冲突」与既有错误码约定冲突

- **哪部分有问题**：`proposal.md:30`。
- **后果**：本仓已有 `VERSION_CONFLICT`（`errors-http.ts:36` → 409）并且注释写明「同键不同摘要属于请求冲突，不是服务器故障」。提案选择了「静默无限重试到成功」，即**刻意规避**这个约定。这会让 409 这条已经在用的运维信号在这个功能上完全消失——出问题时监控看不到任何 4xx 抬升，只看到 CPU 打满。
- **建议**：保留 409 作为「重试耗尽」的出口，让 409 率成为这个功能的核心 SLI。这比「用户感知不到」更符合运维需要——用户感知不到的故障，运维也感知不到。

---

## 结论

| 级别 | 问题 | 关键位置 |
|---|---|---|
| P0-1 | 无界同步重试循环，钉死事件循环（5s×N） | `proposal.md:11-24` + `db.ts:8-9,21` |
| P0-2 | 读-判-写无事务，冲突窗口自造 | `proposal.md:12-21` + `tx.ts:1-12` |
| P0-3 | 表/字段/迁移文件不存在，与 folder 模型不符 | `proposal.md:9-13` vs `0012:33-38`、`0019:26-30`、`0039` |
| P0-4 | `SQLITE_BUSY` 绕过 CAS，三档降级未定义 | `proposal.md:26-30` + `db.ts:59` |
| P1-1 | 无频控/无前端合并，UI 频率打 DB | `proposal.md:4` + `rate-limit.ts:57-66` |
| P1-2 | 零可观测性，health/日志均无覆盖 | 全文 + `health.ts:1-10` |
| P1-3 | 无 `UNIQUE(folder_id,position)`/CHECK，可产生脏序 | `proposal.md:9` |
| P2-1 | 回滚与发布顺序未定义 | 全文 + `migrate.ts:70-79`、`0042:15-17` |
| P2-2 | 静默重试抹掉 409 信号 | `proposal.md:30` + `errors-http.ts:36` |

**放行条件（必须全满足）**：① 改为「drop 时整批提交 + 单事务 IMMEDIATE + 有界重试（≤3，含退避与 deadline）」；② 补齐 ≥0043 迁移文件，含 `folder_id` 维度约束与索引替换；③ 定义 `changes=0` / `SQLITE_BUSY` / 重试耗尽三档行为并把后二者映射到 409；④ 加 `favorite_reorder` 频控档 + 重试计数日志。

**未读到的部分（如实说明）**：我没有找到本功能对应的路由实现与前端代码（`src/app` 仅列为目录，未逐一展开），因此「前端每次拖拽发一个请求」这一句我只能按提案原文采信，未做代码级核实。