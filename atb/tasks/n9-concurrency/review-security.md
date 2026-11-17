# 评审：收藏夹条目排序并发更新 · 安全工程师视角

> 评审对象：`atb/tasks/n9-concurrency/environment/proposal.md`（34 行）
> 评审人身份：**安全工程师**（只看可利用性、越权与资源耗尽、数据完整性、失败时的安全默认；不评 UI 与排序算法体验）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**方案本身一个 `while (true)` 就能被单个已登录用户当作「服务端 CPU 与数据库写锁耗尽的开关」，
> 而且它的 SQL 没有任何一层把「这个 position 属于谁」钉死——乐观锁被当成了授权检查。**

---

## 摘要（前 5 条最重要）

1. 【严重】§2 的 `while (true)` 无上限、无退避 = 可被**单个账号**触发的活锁与事件循环阻塞。
   依据：`proposal.md:17-25,28-29`；`src/server/platform/db.ts:6-8`（better-sqlite3 同步驱动，「等待会阻塞所属线程」）、`:56`（`busy_timeout = 5000ms`）；
   `src/server/worker/handlers.ts:284`（仓库把「DB busy」列为**可重试故障**，说明它是真实发生过的失败模式）。
   后果：攻击者用同一账号对同一 `(user_id, plugin_id)` 并发发 100 个请求，每个请求的循环互相把对方的 `version` 顶掉；
   每一次失败都是一次 `SELECT + UPDATE` 的**同步**往返，全部跑在同一个 Node 事件循环上——**其他人的请求一起排队**。
   这是最廉价的一种 DoS：不需要内网、不需要越权，只要登录。
   建议：**根本不该有循环**（见摘要 2）。若保留乐观锁，必须有 `MAX_RETRIES`（如 3）+ 指数退避 + 抖动，
   且超过上限返回 `409 conflict` 让客户端重试，而不是服务端死转。

2. 【高】单行更新根本不需要版本号：`PRIMARY KEY (user_id, plugin_id)` 已经把并发冲突消灭了。
   依据：`migrations/0012_interactions.sql:32-37`（`plugin_favorites` 的主键就是 `(user_id, plugin_id)`）；
   `proposal.md:18-23` 的 SQL 本来就带 `WHERE user_id = ? AND plugin_id = ?`——同一行、同一用户。
   后果：这里不存在「两个写入者竞争同一行的不同字段」的场景，`version` 只是把最后一写获胜的覆盖问题，
   换成了「大量请求在循环里互相踩」的性能与可用性问题。**安全视角看，这个 version 唯一增加的属性就是可被用来制造冲突。**
   建议：直接 `UPDATE plugin_favorites SET position = ? WHERE user_id = ? AND plugin_id = ?`（单语句、原子、无循环）；
   若真想防「基于旧顺序的脏写」，用**客户端带回的整批顺序 + 服务端一次事务**（见摘要 4），而不是逐行版本号。

3. 【高】§1 的 `position` 没有任何归属约束：同一收藏夹内可以出现重复/任意 position，排序结果可被操纵。
   依据：`proposal.md:11-12`（只加了 `position INTEGER NOT NULL DEFAULT 0`）；
   仓库同类位置信息的既有做法是「作用域唯一索引」：`migrations/0019_favorite_folders.sql:23-24`（`uq_favorite_folders_default ... WHERE is_default = 1`）。
   后果：① 所有历史收藏的 `position` 都是 0（`DEFAULT 0`），**升级瞬间整个收藏夹的顺序变成未定义**，`ORDER BY position` 结果取决于 SQLite 的行序；
   ② 攻击者可以把自己的收藏全部设成 `position = 0` 或极大值，把别人的**展示顺序**搅乱（这不是越权写，但属于可操纵的共享状态）；
   ③ 没有 `UNIQUE(folder_id, position)`，即使正常使用也会因为并发写入产生并列值。
   建议：`position` 用「稀疏整数/第 N 位的 (position, plugin_id, user_id) 复合排序键」而不是要求全局唯一；
   若要唯一，必须 `UNIQUE(folder_id, position)` **且**在移动时于一个事务内做「腾位 → 落位」，否则唯一约束会让并发写入互相拒绝；
   迁移里必须给历史行按现有排序（`created_at DESC, plugin_id`，见 `0012:40`、`0019:29-30` 的索引）回填 position，而不是留全 0。

4. 【高】没有任何「整批」语义：拖拽的真实操作是「一次重排整个夹」，方案却逐条更新，导致部分成功。
   依据：`proposal.md:4-5`（「批量更新」）、`:17-25`（一条 SQL 只改一个 plugin）；
   仓库已有的事务封装：`src/server/platform/tx.ts:1-19`（「跨模块流程创建一次短事务」「不得各自提交独立事务后假装整体原子成功」）；
   同类批量接口的既有写法：`src/app/api/me/favorites/batch/route.ts:15-42` + `src/server/modules/interactions/index.ts` 的 `batchFavoriteAction`。
   后果：拖拽 20 个条目 = 20 个独立请求 = 20 个独立事务；中途失败/被限流/断网，**收藏夹停留在半个新顺序 + 半个旧顺序**，
   这是**持久的数据损坏**（用户无法自愈，只能手动再拖一次）。安全视角：数据完整性被部分失败的写路径破坏。
   建议：`POST /api/me/favorites/order` 一次性提交 `{ folderId, items: [{pluginId, position}] }`，在**一个** `withTransaction` 内批量 UPDATE；
   事务内先校验「这批 pluginId 全部属于该用户该夹」（一条 `SELECT ... WHERE user_id = ? AND folder_id = ? AND plugin_id IN (...)`），
   数量与参数不一致就整体拒绝（照 `batchFavoriteAction` 的既有风格）。

5. 【中】鉴权只靠 `user_id = ?` 出现在 WHERE 里，方案没有一句话说明「userId 从哪来」。
   依据：`proposal.md:18-23`（`userId` 是黑盒参数）；仓库要求是**从会话上下文取，绝不从请求体取**：
   `src/server/platform/context.ts:27-45`（`assertCanWrite` = 登录 + 账户 active）；`src/app/api/me/favorites/batch/route.ts:18-20`（先 `assertCanWrite` 再读 body）。
   后果：如果实现者把 `userId` 当成 body 参数（方案没写但它长得就像），就是**水平越权**：改任意用户收藏夹的顺序；
   而 `WHERE user_id = ?` 这条「天然防线」只在参数可信时才成立——安全不能依赖调用方自律。
   建议：接口签名固定为「服务端从 `ctx.userId` 取」，body 只接受 `folderId / items`；
   并在写入前 `assertFolderOwned(conn, userId, folderId, 'plugin')`（仓库已有同名函数，`src/server/modules/interactions/index.ts:200-209`）。

---

## 完整问题清单（按严重度）

### S-01 【严重】`while (true)`：无上限重试 = 单账号可触发的服务端活锁
- 现场：`proposal.md:17-25,28-29`。
- 依据：`src/server/platform/db.ts:6-8,56`；`src/server/worker/handlers.ts:284`；`src/server/modules/tasks/constants.ts:39,59-64`（仓库对重试的一贯做法是**有限次数 + 退避**，连后台任务都只有 3 次）。
- 后果：见摘要 1。补充：`busy_timeout = 5000ms` 意味着一旦发生写锁争用，**每个等待者的单次调用就可能阻塞 5 秒**；循环里再乘以次数。
- 建议：删除循环；必须保留时加 `maxAttempts=3` + 退避 + 超限 `409`。

### S-02 【高】把 `version` 当授权检查，但它只是并发控制
- 现场：`proposal.md:12,22-23`。
- 依据：`migrations/0012_interactions.sql:36`（主键 `(user_id, plugin_id)`）。
- 后果：见摘要 2。安全影响的具体形式：**任何人只要能制造 version 变化，就能让你的更新失败**（跨用户不成立，但同账号多标签页/多设备成立；配合 S-01 就是自伤式 DoS）。
- 建议：去掉 version；需要防脏写就用整批提交 + 事务（摘要 4）。

### S-03 【高】`position` 无作用域约束与回填，升级即乱序、可被操纵
- 现场：`proposal.md:11`。
- 依据：`migrations/0019_favorite_folders.sql:23-24`（作用域唯一索引的先例）；`0012:40`、`0019:29-30`（既有排序用的是 `created_at DESC, plugin_id`）。
- 后果：见摘要 3。
- 建议：迁移里按 `created_at DESC, plugin_id` 回填 position；排序用 `(position, plugin_id)` 兜底；若要唯一，用「事务内腾位」。

### S-04 【高】逐条更新 = 部分成功 = 持久的数据损坏
- 现场：`proposal.md:4-5,17-25`。
- 依据：`src/server/platform/tx.ts:1-19`；`src/app/api/me/favorites/batch/route.ts:15-42`；`src/server/modules/interactions/index.ts` 的 `batchFavoriteAction`（已有批量接口范式与「有一项不属于本人则整批拒绝」的语义）。
- 后果：见摘要 4。
- 建议：整批 + 单事务 + 先校验归属。

### S-05 【中】鉴权来源未定义，存在水平越权风险面
- 现场：`proposal.md:18-23`。
- 依据：`src/server/platform/context.ts:27-45`；`src/app/api/me/favorites/batch/route.ts:18-20`；`src/server/modules/interactions/index.ts:200-209`（`assertFolderOwned`）。
- 后果：见摘要 5。
- 建议：userId 只从 ctx 取；`assertCanWrite` → `assertFolderOwned` → 批量 UPDATE，同一事务。

### S-06 【中】没有频控：拖拽接口可被用作批量写放大
- 现场：`proposal.md` 全文无频控。
- 依据：仓库对写接口一律挂频控：`src/server/platform/rate-limit.ts:46-55,68-134`（9 个动作）、典型用法 `src/app/api/me/roles/applications/route.ts:41-56`（`enforceRateLimit` + 失败 `refundQuietly`）。
- 后果：一次拖拽生成 N 个请求、每个请求可能进 N 次循环 → 写放大叠加；单账号即可把 SQLite 写锁打满。
- 建议：新增 `favorite_order` 动作（如 30 次/分钟/用户），失败退还额度；配合整批接口后请求数从 N 降到 1。

### S-07 【低】失败语义未定义：客户端无法区分「冲突」与「无权限」
- 现场：`proposal.md:28-29`（「循环重试，直到成功」）。
- 依据：`src/packages/common/errors.ts` 的 DomainError 体系（`err.conflict` / `err.forbidden` 等），HTTP 层按错误码映射状态码（`src/server/app/api/_lib/context` 的 `errorResponse`）。
- 后果：如果 `row` 查不到（收藏已被别处删除），方案里的 `row.version` 会**直接抛 TypeError**（`proposal.md:23` 对 `undefined` 取值），
  被折成 500——把「已取消收藏」这种正常情况报成服务器故障。
- 建议：`row` 为空时 `err.unavailable('该收藏不存在或已被取消。')`；冲突 `err.conflict`；越权 `err.forbidden`。

---

## 结论与建议

**按现方案实现会同时引入一个「自伤式 DoS 开关」和一个「部分成功的数据损坏路径」**，而它想解决的问题（顺序错乱）
在 `(user_id, plugin_id)` 主键 + 整批事务提交下**根本不存在**。

**建议的最简安全实现**：
1. `ALTER TABLE plugin_favorites ADD COLUMN position INTEGER NOT NULL DEFAULT 0;` 并在迁移里按 `created_at DESC, plugin_id` 回填。
2. `POST /api/me/favorites/order`，body `{ folderId, items: [{ pluginId, position }] }`；
   `assertCanWrite(ctx)` → `assertFolderOwned(...)` → 一个 `withTransaction` 内校验归属 + 批量 UPDATE。
3. 不引入 `version`、不写循环；并发语义是「最后一次整批提交获胜」，这对拖拽是正确语义。
4. 加频控（`favorite_order`，30 次/分钟），失败退还额度。
5. 排序读路径用 `ORDER BY position ASC, plugin_id ASC`，保证并列时结果稳定。

> 评审人声明：本报告只覆盖安全/完整性/可用性视角。排序算法与前端交互体验由其他视角负责；
> 但 S-01 与 S-04 是**不论排序怎么设计都必须解决**的两条。

> 前提核实：`plugin_favorites` 与其 `(user_id, plugin_id)` 主键真实存在（`migrations/0012_interactions.sql:32-37`），
> `folder_id` 与既有排序索引也在（`0019:26-30`）；方案要新增的 `position` / `version` 两列是**声明的 ALTER**（合理）。
> 但方案声称的「并发时会出现顺序错乱」在本仓库当前 schema 下**没有代码证据**——现有收藏写入全走 `withTransaction`（`interactions/index.ts` 内 25 处），
> 真正的不一致来源更可能是「逐条请求导致部分成功」，即 S-04。
