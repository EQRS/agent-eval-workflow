# 评审：事务封装的 async 化改造 · 安全工程师视角

> 评审对象：`atb/tasks/n57-transaction-wrapper/environment/proposal.md`（41 行）
> 评审身份：**安全工程师**（只看原子性/一致性不变量、越权面的时序、失败语义、审计与检索索引完整性；不评代码风格）
> 结论：**这个改造会把「事务」从一个安全边界变成一个摆设。** `proposal.md:23-33` 的 `db.exec("BEGIN") → await fn(tx) → COMMIT`
> 会让出事件循环，事务在「未提交」状态下被挂起，其他请求在同一个连接上继续写——原子性、审计一致性、检索索引一致性全部失去保证。
> 现状 `withTransaction` 的同步签名本身是**刻意的安全设计**（`src/server/platform/tx.ts:37-52` 注释明写「传入的回调**必须是同步函数**」），改造属倒退。

---

## 摘要（前 5 条）

1. 【严重】`await fn(tx)` 让出事件循环 → 事务边界被其他请求穿透，原子性直接被破坏。
   依据 `proposal.md:23-33`；`src/server/platform/db.ts:5-8`（「better-sqlite3 是同步驱动，等待会阻塞所属线程」）。
2. 【严重】`exec("BEGIN")` 与 `exec("ROLLBACK")` 绕过 better-sqlite3 的事务管理：嵌套即抛错，回滚即越界回滚外层。
   依据 `proposal.md:24,27,30`；`src/server/platform/tx.ts:43-51`（用 `db.transaction()`，嵌套转 SAVEPOINT）。
3. 【高】审计与业务不再同一事务，审计退化成「尽力而为的日志」。依据 `src/server/modules/operations/service.ts:161-174`、`migrations/0042_audit_immutable.sql:1-15`。
4. 【高】全量改写调用点（33 个文件 / 161 处）没有验收与回滚路径，鉴权、可见性、撤回这些安全敏感路径都在其中。
5. 【高】`COMMIT` 抛错后不强制回滚 → 连接上残留活动事务，「上一位用户的断言」泄漏成「下一位用户的前置条件」。依据 `proposal.md:27`；`src/server/platform/db.ts:50` + `migrations/0001_baseline.sql:109-111`（延迟外键在 COMMIT 才校验）。

---

## P0

### 1. 事务在 `await` 处被挂起：别的请求在同一连接上「钻进」未提交事务
- **哪部分有问题**：`proposal.md:23-33`——`db.exec("BEGIN")` 之后 `await fn(tx)`。只要内部有真实的让出点（`await fetch` / `http` / `promisify`），进程内其它请求就会继续在**同一个 `db` 单例连接**上执行语句。
- **后果（文件:行号）**：
  - `src/server/platform/db.ts:66-74`：`getDb()` 是**进程级单例连接**，事务状态是连接级的，没有第二连接可隔离。
  - `src/server/platform/db.ts:5-8`、`src/server/platform/tx.ts:37-41`：注释明写「同步驱动」「回调必须是同步函数」。
  - 具体污染路径：`src/server/platform/rate-limit.ts:229`（`limiter.consume()` 对同一 `db` 写）若落在别人挂起的 async 事务里，对方 ROLLBACK 会**抹掉这次频控消耗**——被限流的用户重新拿到额度（同源 `rate-limit.ts:14-23` 的「单条语句 + 事务包装」正是靠原子性）。
  - 同类：`src/server/modules/identity/service.ts:278-313`（`touchSession` 续期 UPDATE）与 `:301`（`last_seen_at` 写入）会被回滚，会话滑窗续期失效。
  - 这是**安全后果**不是性能后果：限流、会话、审计这些旁路状态一旦共享事务边界，回滚语义对它们失控。
- **建议**：保留 `(fn: (tx: Tx) => T): T` 同步签名；需要「事务前先异步准备」的流程沿用现状（`src/server/modules/comments/index.ts:307-315`：先在事务外 `await loadEnqueue()`，再进同步事务）。

### 2. 手写 `BEGIN/COMMIT/ROLLBACK` 绕过驱动的事务管理：嵌套抛错 + 越界回滚
- **哪部分有问题**：`proposal.md:24` `db.exec("BEGIN")`、`:27` `COMMIT`、`:30` `ROLLBACK`。
- **后果（文件:行号）**：
  - `src/server/platform/tx.ts:43-51`：现有实现用 `db.transaction()`（内建 SAVEPOINT 嵌套 + 异常自动回滚）。全仓 **33 个文件、161 处** `withTransaction` 调用（本轮 grep 统计）。
  - 嵌套路径早已存在：`src/server/modules/operations/service.ts:176-184`、`src/server/workflows/publish-review.ts:150-165`（发布事务内写审计）、`src/server/modules/comments/index.ts:317`+`:417`（事务内调 `recordOperationInTx` 后又入队）。
  - ① 这些路径都会抛 `cannot start a transaction within a transaction` → 新封装在嵌套场景**完全不可用**。
  - ② 更危险：外层已有事务时内层 `BEGIN` 失败，catch 仍要执行 `ROLLBACK`（`proposal.md:29-31`），**把外层事务一起回滚**。外层以为自己原子提交成功，实际已丢写——静默数据丢失路径。
  - ③ `PRAGMA` 在事务内是空操作（`src/server/platform/migrate.ts:34-36` 明确记载），手写 BEGIN 的封装与迁移动线交互时会让 `foreign_keys` 开关静默失效。
- **建议**：不手写 BEGIN/COMMIT；用 `db.transaction()`（已实现 SAVEPOINT 嵌套与自动回滚），或索性沿用现状只补一个事务上下文对象。

### 3. `COMMIT` 失败不回滚 → 连接上残留活动事务，「上一位用户的断言」泄漏成「下一位用户的前置条件」
- **哪部分有问题**：`proposal.md:26-28`——`const result = await fn(tx); db.exec("COMMIT"); return result;` 只有一次 try/catch，COMMIT 自身失败时没有二次 ROLLBACK 契约，也没有「连接是否仍在事务中」的清理。
- **后果（文件:行号）**：
  - `src/server/platform/db.ts:50`（`defer_foreign_keys = ON`）+ `migrations/0001_baseline.sql:109-111`（`DEFERRABLE INITIALLY DEFERRED` 外键）：**外键校验发生在 COMMIT**，所以 COMMIT 是这条链上最可能抛错的一步（`SQLITE_BUSY` 见 `db.ts:56` 的 5000ms）。
  - 连接残留事务时，下一位用户 B 的所有语句都落进这个「幽灵事务」；A 的异常路径可回滚 B 的写。对幂等审计（`migrations/0001_baseline.sql:411-433` 的 `UNIQUE(action_type, operation_key)`）而言，去重行可能凭空消失 → 同一操作可重放。
  - 现有实现无此问题：`db.transaction()` 由驱动保证「异常必回滚、事务必结束」。
- **建议**：继续由驱动管事务；若坚持手写，COMMIT 失败后必须无条件尝试 `ROLLBACK`，并在仍失败时把连接标记为不可用——但仓内是单例连接（`db.ts:66-74`）没有丢弃重连通道，这本身就是「不该手写」的论据。

---

## P1

### 4. 审计与业务不再同事务：审计从「证据」降级为「尽力而为的日志」
- **哪部分有问题**：整个封装（`proposal.md:23-33`）默认「回调内的动作属于同一事务体」，一旦拆成多个 `await` 段，跨段一致性不再由 SQLite 保证。
- **后果（文件:行号）**：
  - `src/server/modules/operations/service.ts:161-174`：`recordOperationInTx` 的存在理由就是「审计必须与业务变更同事务（架构 §6.4）」。
  - `migrations/0042_audit_immutable.sql:1-15`：审计表加固的原因正是「删除审计行即可让已消费的 `operation_key` 重新可用，构成**不需要改代码的伪造通道**」——事务一致性一松，同样效果可由**失败路径**取得。
  - `src/server/workflows/publish-review.ts:119-165`：发布 = 改公开指针 + `reindexReviewInTx` + `recordOperationInTx` 三段同事务。async 化后若中间让出，管理员隐藏/撤回（`content_entities.hidden_at`，`:120`）可插入，发布基于过期快照提交 → **已隐藏内容被发布**。
  - `src/server/modules/operations/service.ts:97-113`（同键「先查后插」的幂等）在让出点被并发穿透：同一次操作写两条或零条审计。
- **建议**：保留「审计 + 业务 + 索引同同步栈」；把需要异步的步骤（外部核验、Markdown 解析、图片处理、动态 import）全部移到事务之前（`tx.ts:6-7`、`comments/index.ts:307-315` 已是该模式）。

### 5. 越权判定与写入之间插进让出点：禁用/降权账号仍能完成写入（TOCTOU）
- **哪部分有问题**：`proposal.md:23-33` 的 `tx` 在 `await` 期间仍然有效；而仓内鉴权是**刻意放在同步事务内**，依赖「断言与写入不可分割」。
- **后果（文件:行号）**：
  - `src/server/platform/context.ts:27-43`：注释写明「角色与账户状态**每次请求从数据库读取**……撤销管理员后其普通登录会话不因此获得旧管理权限」。async 事务里断言在 `await` 前、写入在 `await` 后 → 中间被禁用/降权的账号仍完成写入。
  - `src/server/modules/packs/index.ts:1592-1605`、`:706-731`：`assertCanWrite(ctx)` 紧接 `withTransaction`，事务内再读 `withdrawn_at/hidden_at` 再写；`src/server/modules/interactions/index.ts:1-10`（纪律 1「写入之前先确认目标公开且存在」）+ `src/server/modules/content/index.ts:367`（`resolveVisibilityInTx`）。
  - 反向也成立：若鉴权留在事务前的 `await`，事务先开读栈、写锁在 wait 中途升级，撞 `db.ts:56` 的 5000ms `busy_timeout`，表现为偶发 500，错误路径又回到第 3 条的残留事务。
- **建议**：不改事务模型。长操作放到事务外并显式建模状态机（`content_submissions.processing_state` + `state_version` 乐观锁，`publish-review.ts:110,139` 已在用）。

### 6. 全量改写 33 个文件 / 161 处，且本仓的同步返回值契约没有迁移面
- **哪部分有问题**：`proposal.md:36-37`「把所有 `withTransaction` 调用点改成 await 风格」。
- **后果（文件:行号）**：
  - `src/server/platform/tx.ts:42,55`：签名是 `<T>(...): T`。改成 `Promise<T>` 后，所有「结果被赋值给静态类型 / 立即解构」的调用点都要改：`src/server/modules/roles/service.ts:498`（`const view: MyRolesView = withRead(...)`）、`src/server/modules/comments/index.ts:317,469`、`src/server/modules/moderation/service.ts:313`、`src/server/workflows/submit-review.ts:193`、`src/app/api/forum/attachments/route.ts:48`、`src/app/api/forum/drafts/[draftId]/route.ts:103`。
  - 服务层同步方法必须整体 `async`，一路冒泡到 API handler——**全站签名迁移**，却没有给验收口径。
  - 安全侧：漏改一处就把「同步完成」变成「Promise 未等待」，在 `void` 上下文里 TypeScript 不一定报错，运行期表现为**写入尚未发生就返回 200**——伪造成功响应，正是 `docs/项目规划/README-切片说明.md:208` 明令禁止的「假装成功」。
- **建议**：不迁移签名。若只想「await 手感」，在调用点 `await` 一个已 resolve 的同步结果即可，不动 `tx.ts`。

---

## P2

### 7. 丢掉「事务已结束」保护：越界访问从「响亮失败」变成「静默自动提交」
- **哪部分有问题**：`proposal.md:23,26`——`tx` 作为普通对象透传，没有 `finished` 状态跟踪。
- **后果（文件:行号）**：`src/server/platform/tx.ts:21-35`：`makeTx()` 的 getter 在 `markFinished()` 之后访问 `tx.db` 会直接抛错。方案把 `tx` 泄漏出去后（闭包、队列、`setTimeout`），语句在**没有活动事务**的连接上执行 = SQLite 逐条自动提交，本该被回滚的写变成已生效——审计/可见性路径上的数据完整性事故。
- **建议**：任何新封装都必须复用 `makeTx`，不得直接透传 `db`。

### 8. 封装不接受连接参数，摧毁「连接一律走工厂」的单一边界
- **哪部分有问题**：`proposal.md:23`——`transaction(fn)` **没有 `db` 参数**，与现状 `withTransaction(fn, db)` 不等价。
- **后果（文件:行号）**：`src/server/platform/db.ts:10`（「其他模块不得自行 `new Database(...)`」）、`src/server/platform/tx.ts:8`（「页面与 HTTP 入口不能自行指定连接或事务对象」）确立的是**单一入口**；而 `tx.ts:42` 的第三参数是测试与多库场景的注入点（`tests/forum-misc.test.ts:467,493,512`、`tests/drafts.test.ts:11` 的注释即讲这个注入点）。砍掉参数后，测试会悄悄跑在默认库 `.data/app.sqlite`（`db.ts:33`），或被迫另开连接导致事务边界分裂。
- **建议**：保留 `db` 参数并加品牌类型（branded `SqliteDatabase`），确保只能来自 `openDatabase`。

### 9. 持久性与 PRAGMA 的可观测性缺口
- **哪部分有问题**：`proposal.md:23-33` 完全没提 PRAGMA 与持久性；而 async 事务会显著拉长连接被占用的时间。
- **后果（文件:行号）**：`src/server/platform/db.ts:48-56` 在每条连接上设 `foreign_keys`/`defer_foreign_keys`/`journal_mode`/`synchronous=FULL`/`busy_timeout`；`synchronous` 在**全仓仅此一行**出现，`src/server/platform/migrate.ts:239-245` 与 `src/server/platform/health.ts:67-75` 的自检**不查 `synchronous`**。也就是说持久性被悄悄降级（或被这条改动顺带改掉）时，健康检查看不出来——运维只看到「偶发 500」，安全侧则看到「审计偶尔缺行」。
- **建议**：把 `synchronous` 纳入 `selfCheck`（`migrate.ts:239`）与健康报告；事务层不碰 PRAGMA。

### 10. 没有并发/失败注入的验证口径
- **哪部分有问题**：`proposal.md:39-41`「预期：代码更直观 / 支持异步操作」——零验收。
- **后果（文件:行号）**：`tests/forum-misc.test.ts:459-519` 是全仓唯一覆盖事务语义的用例（`:480-499` 测回滚一致性）；`tests/` 84 个文件里只有 2 个触及 `withTransaction/withRead`。它们断言的是「同步提交后的最终状态」，对「中途让出」零覆盖——任何 async 化都能让这些用例继续绿。
- **建议**：若仍要推进（不建议），必须同时交付：① 双请求交错注入测试、② COMMIT 失败注入测试、③ 用 `scripts/verify-tests-bite.mjs` 做变异验证，证明用例在改动被回退时真会变红。

---

## 结论

不建议采纳。前提（「回调写法不够直观」）成立，但代价是拆掉本仓最关键的一条安全边界——同步事务：

| 安全不变量 | 现状锚点 | 本方案后果 |
|---|---|---|
| 业务写原子（全成或全无） | `src/server/platform/tx.ts:43-51` | 被其它请求穿透 |
| 审计与事实同事务 | `src/server/modules/operations/service.ts:161-174`、`migrations/0042_audit_immutable.sql` | 审计可丢 → 去重键重生 |
| 可见性判定与写入同事务 | `src/server/modules/interactions/index.ts:1-10`、`src/server/modules/content/index.ts:367` | 撤回/隐藏可被穿透 |
| 权限判定与写入不可分割 | `src/server/platform/context.ts:27-43` | 降权/禁用后可完成写入 |
| 失败必回滚、事务必结束 | better-sqlite3 `db.transaction()` | COMMIT 失败留下幽灵事务 |

若要「await 手感」，正确做法是**在事务外等待、在事务内同步**（现状已如此），而不是把事务变成一个可挂起的对象。
