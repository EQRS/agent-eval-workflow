# n57-transaction-wrapper · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0019
> 模型 deepseek-chat · 5116 字 · 24 次工具调用

---

# SRE 评审：n57「把事务封装成 async 函数」

已实际读取：`proposal.md`（41 行）、`src/server/platform/tx.ts`、`db.ts`、`migrate.ts`、`heartbeat.ts`、`modules/operations/service.ts`，以及 `migrations/0001/0002/0004/0021/0042`。

**结论：该方案在当前技术栈上不可实施（不可只靠改代码实现），必须终止；它的每一项根本假设都被仓库事实否定。**

---

## 一、致命级（阻塞发布，SRE 一票否决）

### P0-1 `db.exec("BEGIN")` 与连接 PRAGMA 策略直接冲突，且不经 better-sqlite3 事务栈

- **① 问题部分**：`proposal.md:24-34` 手写 `db.exec("BEGIN")` / `COMMIT` / `ROLLBACK`。
- **② 后果（文件:行号）**：
  - `db.ts:57-59` 调用 `db.pragma('defer_foreign_keys = ON')`，这**仅对单个事务生效**，事务结束即复位。better-sqlite3 的 `db.transaction()` 内部在每次 BEGIN 前重新应用它；走裸 `db.exec("BEGIN")` 不经过该路径，导致 `defer_foreign_keys` 失效，于是 `0021_content_entities_pack_release.sql:38-41` 的 `DEFERRABLE INITIALLY DEFERRED` 外键在 `content_entities` ↔ `reviews` / `plugin_pack_releases` 的配对写入中变成**立即校验**。插入顺序稍有变化就抛 `FOREIGN KEY constraint failed`，这是**发布即挂**，不是概率问题。
  - `db.ts:47-48, 62-66` 有外键启用断言，但**没有**“不在手写事务内”的断言，裸 BEGIN 不会触发任何保护。
  - `tx.ts:40-50` 的现有实现用 `db.transaction()`，是唯一能正确维护 `defer_foreign_keys`、嵌套 savepoint、异常时自动回滚的路径。方案把它换掉了。
- **③ 建议**：禁止任何 `db.exec('BEGIN'|'COMMIT'|'ROLLBACK')` 出现在 `src/**`（`migrations/*.sql` 除外，那是迁移器的合法职责，见 `migrate.ts:47-56`）。若确需新 API，应在 `tx.ts` 内包 `db.transaction()`，不得绕过。

### P0-2 异步回调 + 同步驱动 = 事务跨 await 悬挂，长时间持有写锁

- **① 问题部分**：`proposal.md:23` 签名 `fn: (tx: Tx) => Promise<T>`，`proposal.md:30` `await fn(tx)`，即事务内存在 await 点。
- **② 后果（文件:行号）**：
  - `db.ts:9-11` 明确写死纪律：“better-sqlite3 是同步驱动，等待会阻塞所属线程，因此业务事务保持短小”。一旦 `await` 让出事件循环，事务处于 **open 状态但无人执行**，而 `BEGIN` 已可能持有写锁。
  - `db.ts:60` `busy_timeout = 5000`：`db.ts:66-70` 说明 Web 与 worker 是**两个独立进程**（`getDb()` 每进程一个 singleton）。跨进程写锁争用下，第二个进程 5s 超时后抛 `SQLITE_BUSY`。在事务内 await 一个网络或文件操作（`tx.ts:8` 明令“外部拉取、Markdown 解析、图片处理、文件复制一律在事务之外”），持锁时间从毫秒级升到**秒级**，`RateLimit`、`heartbeat`（`heartbeat.ts:70-77`，每 5s 必须写入一次）会大面积 `SQLITE_BUSY`。
  - 连带 SRE 后果：`heartbeat.ts:26-31` 中 `WORKER_STALE_MS=30s`、`WORKER_DEAD_MS=120s`。worker 心跳写不进去时，健康检查会把活着的进程判成 `stale`/`dead`，触发容器重启 → 重启又持锁 → **级联重启循环**。这类故障在日志里往往只有一个 `SQLITE_BUSY`，正是 `heartbeat.ts:6-8` 描述的最难排查场景。
  - `tx.ts:20-22` 的 `finished` 守卫假设“回调返回后事务即刻结束”。在 async 下，`finally { tx.markFinished() }`（`tx.ts:47`）在 await 完成前不会执行，`finished` 长期为 false，**“事务结束后禁止写入”这道防线实际上被架空**。
- **③ 建议**：保留“回调必须是同步函数”的硬约束。若业务确实需要异步，正确做法是**把 await 提到事务外**（先拉取/解析，再进同步事务落库），而不是放宽事务接口。可加运行时断言：若 `fn` 返回 Promise 则立即抛错，防止后续开发者误用。

### P0-3 改造范围 = 全站写路径，无灰度、无回滚演练，且审计去重不变量会漂移

- **① 问题部分**：`proposal.md:38` “把所有 withTransaction 调用改成 await 风格”。
- **② 后果（文件:行号）**：
  - `tx.ts:44-51` 是**全站唯一**写事务入口，被 `modules/operations/service.ts` 等 22 个模块目录引用（`src/server/modules/` 下 22 个子目录）。一次性全量改造意味着**单次发布覆盖全部写路径**。
  - 最危险的是审计去重。`0001_baseline.sql:426` 有 `UNIQUE (action_type, operation_key)`，`0042_audit_immutable.sql:5-7` 说明 `setUserRole`/`setUserStatus`/`configuration_revisions`/catalog 去重**全依赖**这条唯一约束。`operations/service.ts:139-145`（`recordOperationInTx`）的注释写得很明确：`record()` 是 async 接口，“在同步事务回调里不能 await”。一旦事务变 async，调用方**天然会想去 await record()**，而 `record()` 内部 `service.ts:148-151` 在 `input.tx` 缺失时会**自己开一个新事务**——在 async 事务内嵌套 `db.transaction()` 会退化成 savepoint 或直接抛错，两种结果都破坏“审计与业务变更同事务”。成功路径上表现为审计丢失或去重失效（重复扣减、重复角色变更）。
  - 该失败是**静默的**：外键和审计触发器都拦不住“审计行写到了另一个事务里”。
- **③ 建议**：即便方案修复，也必须分批：先加双轨（新旧并存、按模块开关），跑一轮影子对比（同请求双跑，比对落库行数与 `operation_records` 内容），再逐模块切换。附回滚预案：单开关一键退回 `withTransaction`，不依赖回滚数据库。

---

## 二、严重级

### P1-4 方案未涉及 `PRAGMA foreign_keys` 的事务边界，与迁移器约定存在冲突风险

- **① 问题部分**：`proposal.md` 全文未提 `foreign_keys`/`defer_foreign_keys` 的事务语义。
- **② 后果（文件:行号）**：
  - `0021_content_entities_pack_release.sql:2` 用 `-- dsh:migration-transaction=off`，`migrate.ts:47-56` 明确说明原因：**`PRAGMA foreign_keys` 在事务内是 no-op**，重建被外键引用的表必须先关外键。方案引入“手写 BEGIN”的范式后，很容易被后来者误用到迁移或维护脚本里，产生**半成品表**（`migrate.ts:54-55` 原文：“中途失败会留下半成品”），而这类失败没有事务回滚兜底。
  - `0021:117` 结尾有 `PRAGMA foreign_key_check`，但那是迁移器一次性校验；运行期手写事务若关过外键，**没有任何运行期检测**会发现问题。
- **③ 建议**：在 `tx.ts` 顶部把这条约束写成显式文档，并加 ESLint/CI 规则拦住 `src/**` 里的 `PRAGMA foreign_keys` 与裸 `BEGIN`。

### P1-5 “预期：支持异步操作”是伪收益，真实痛点是可读性

- **① 问题部分**：`proposal.md:41-43` 的收益论证。
- **② 后果（文件:行号）**：`tx.ts:44-46` 的回调确实是同步的，但**唯一**的不便是语法。而现有 `operations/service.ts:139-145` 已经用 `recordOperationInTx` 这个同步入口把问题解决了（“比让每个模块各自内联一份 INSERT 更能保证不变量只写一遍”）。为纯语法糖换取 P0-1/P0-2/P0-3 三类风险，投入产出比不成立。
- **③ 建议**：若目标只是可读性，保留 `withTransaction` 但**允许同步箭头函数返回任意值**（现状已支持，`tx.ts:44` 的 `T` 是泛型），并补一份调用范例文档。真正的 async 需求另开单独立项，走“事务外异步 + 同步落库”的两段式。

---

## 三、SRE 侧还需要补齐的（方案完全缺失）

- **可观测性**：方案没有任何指标。至少需要：事务时长直方图、`SQLITE_BUSY` 计数、`deadlock`/回滚计数，以及按 `action_type` 的审计写入成功率。当前 `heartbeat.ts` 只能看见“进程活着”，看不见“写路径在退化了”。
- **容量与压测**：async 化后事务持有时间不可控，必须给出压测结论（并发写 P99、`busy_timeout` 超时率）才能上线。方案无一字。
- **变更窗口与演练**：涉及全部写路径，需要明确停机窗口或双写过渡期方案，并在预发环境演练“发布中途失败”的回滚。方案无。

---

## 四、建议的处置

| 优先级 | 动作 |
|---|---|
| 立即 | 方案**不予通过**，退回。P0-1/P0-2 属于技术前提错误，不是实现细节。 |
| 短期 | 若要改善可读性：在 `tx.ts` 补 `fn` 返回 Promise 时的**显式拒绝** + CI 规则禁止裸 `BEGIN`/`PRAGMA foreign_keys`。 |
| 中期 | 真正需要异步的场景，按“事务外 await、同步事务落库”模式重构调用点，逐模块推进，带影子对比。 |
| 长期 | 补事务可观测性指标，纳入健康检查（`health.ts` / `heartbeat.ts` 同层）。 |

**关键判断**：`proposal.md:24-34` 这 11 行代码之所以看着合理，是因为它假设了一个异步数据库驱动。而 `db.ts:9-11` 和 `tx.ts:40-43` 两处注释都明确写着本项目用的是**同步**驱动、回调**必须同步**。方案与代码库的根本前提相悖，这是它所有后续问题的来源。