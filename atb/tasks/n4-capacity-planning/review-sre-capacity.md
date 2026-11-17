# 评审：n4-capacity-planning proposal（连接池方案）

结论：**方案方向性错误，不可执行**。它把“SQLite/better-sqlite3 单连接”当成瓶颈，用 Postgres 式连接池去治，
但本仓库的架构根本不是“连接不够用”，而是**同步驱动 + 单进程单线程**。改完之后吞吐不会提升，反而会把外键、事务、WAL、备份四条正确性不变量一起打掉。

> 说明：本评审为只读静态分析，未运行压测与回归测试；所有“后果”均指到具体文件:行号，可复核。

## 一、事实核对（方案的前提是否成立）

| 方案断言 | 仓库实况 | 出处 |
|---|---|---|
| “better-sqlite3 是单连接” | 属实，且是**刻意设计**：进程级单例连接 | [db.ts:28-29](src/server/platform/db.ts#L28-L29)、[db.ts:66-74](src/server/platform/db.ts#L66-L74) |
| “所有请求排队” | **不成立**。better-sqlite3 是同步驱动，不存在 await 排队的连接；并发上限由 Node 单线程 + SQLite 写锁共同决定 | [db.ts:7-8](src/server/platform/db.ts#L7-L8) |
| 引入“连接池提升并发” | 本程序没有需要连接池的并发模型：读请求被同步执行阻塞，池再大也是同一个线程依次跑 | [tx.ts:42-52](src/server/platform/tx.ts#L42-L52) |
| 池大小 = CPU×2+磁盘数 | 该公式是给有阻塞式连接、跨网络 RTT 的 Postgres/MySQL 用的，对本地嵌入式 SQLite 无意义 | proposal.md:26-27 |
| `connectionString: DATABASE_URL` | 仓库里**不存在** `DATABASE_URL`，只有 `DSH_REVIEW_DB_PATH`（文件路径） | [db.ts:33](src/server/platform/db.ts#L33)、[.env.example](.env.example) |
| “吞吐提升 3-5 倍” | 无基线、无度量方法、无验收标准 | proposal.md:33-34 |

## 二、问题清单

### P0-1 把“同步驱动的线程阻塞”误判为“连接池排队”
- **哪部分**：proposal.md:4「现在 better-sqlite3 是单连接，所有请求排队」；proposal.md:20「const conn = await pool.acquire()」
- **后果**：better-sqlite3 的 API **没有异步语义**，`db.prepare().all()` 同步返回，从不 `await`。
  全站事务写法是同步回调：`withTransaction(fn)` 的 `fn` **必须是同步函数**（[tx.ts:37-52](src/server/platform/tx.ts#L37-L52)），`openDatabase` 也是同步的（[db.ts:44-64](src/server/platform/db.ts#L44-L64)）。
  照方案改成 `await pool.acquire()` 意味着每个调用点都变 async —— **142 个文件**（`getDb`/`withTransaction`/`withRead` 调用方）要改签名；
  且 `db.transaction()` 回调一旦掺入 `await` 就会**提交/回滚错乱**：better-sqlite3 事务在回调返回时提交，中途 await 让出的那一刻事务还开着，别的请求会插进来。
  这不是性能优化，是全站数据一致性回归。
- **建议**：删除整个方案。若要提并发，先拿**瓶颈证据**：参照仓库既有的实测方法学（[docs/P3成本基线-执行任务书.md](docs/P3成本基线-执行任务书.md)、[docs/实测记录-M9第三次-经验假设证伪.md](docs/实测记录-M9第三次-经验假设证伪.md)）采 p50/p95 延迟与事件循环延迟，证明瓶颈在 CPU（Markdown 解析 / sharp 图片处理）而非 DB。

### P0-2 连接池会直接摧毁外键与延迟外键不变量
- **哪部分**：proposal.md:10-23（每次 acquire 拿到一条新连接）
- **后果**：每条连接都必须独立执行 `PRAGMA foreign_keys = ON` 与 `defer_foreign_keys = ON`。
  `openDatabase` 靠**建立连接时同步设置并立即读回断言**来保证（[db.ts:47-62](src/server/platform/db.ts#L47-L62)：设完立刻读回 `foreign_keys`，不等于 1 就 close 并抛错拒绝工作）。
  池里的连接是惰性创建与复用的，**没有任何地方能保证借出的那条连接仍处于正确 PRAGMA 状态**；
  一旦漏设，外键就静默失效（`foreign_keys` 关着时不会被报错，只会产生孤儿行）。
  更糟：`defer_foreign_keys` 是**连接级会话状态**，在池里会被污染——一个请求设的延迟外键会泄漏给下一个请求。
- **建议**：真要做多连接，必须做成“连接工厂 + 借出时重放 PRAGMA + 归还时重置”的可证明封装，并为它写断言测试；否则不要碰这条不变量。

### P0-3 事务边界池化后失去“共用同一事务上下文”的保证
- **哪部分**：proposal.md:20-22（`conn.prepare(...)` 后 `pool.release(conn)`）
- **后果**：全仓有 14 处裸 `db.transaction(` 直接用**同一个** db 对象（如 [tasks/service.ts:235](src/server/modules/tasks/service.ts#L235)、
  [notifications/index.ts:586](src/server/modules/notifications/index.ts#L586)、[settings/index.ts:472](src/server/modules/settings/index.ts#L472)）。
  它们内部再调 `getDb()` 必须拿到**同一条连接**，否则就是跨连接提交；
  「公共实体 + 具体子类型」配对依赖的延迟外键（[db.ts:49-50](src/server/platform/db.ts#L49-L50)）在 commit 时要么误报要么漏过。
  另外 `Tx.finished` 这道“事务已结束禁止继续写”的护栏（[tx.ts:24-27](src/server/platform/tx.ts#L24-L27)）绑定的是连接对象，池化后同一对象在请求间流转，护栏失效。
- **建议**：事务上下文必须显式传递（AsyncLocalStorage 或参数透传），不能依赖“全局单例”这一隐含假设。方案对此只字未提。

### P0-4 FTS5 虚表与 35 个触发器是 SQLite 独有资产，“换驱动”会让它们全盘失效
- **哪部分**：proposal.md:10-15（`createPool({connectionString})` 是 pg/mysql2 的写法）
- **后果**：全文检索靠 **FTS5 + trigram 分词**（[migrations/0014_review_search.sql:18-24](migrations/0014_review_search.sql#L18-L24)），
  业务正确性靠 **35 个 BEFORE INSERT/UPDATE 触发器**、分布在 8 个迁移文件（[migrations/0002_triggers.sql:16-30](migrations/0002_triggers.sql#L16-L30)）。
  Postgres 没有等价物：trigram 分词的“1–2 字查询必须走 LIKE 回退”边界（[0014 注释:8-9](migrations/0014_review_search.sql#L8-L9)）会全废，触发器要用 PL/pgSQL 重做 8 个迁移。
  方案自始至终没说“池后面接的到底是不是同一个 SQLite 文件”。
- **建议**：明确写死“池只接同一个 SQLite 文件”，并补“多连接下 FTS5 与触发器行为不变”的验证项。

### P1-1 完全没处理 WAL 与多进程的既有事实
- **哪部分**：proposal.md 全文
- **后果**：系统本来就是**两个进程共用一个库文件**：web（[dsh-review-web.service:19](deploy/dsh-review-web.service#L19)）
  与 worker（[dsh-review-worker.service:17](deploy/dsh-review-worker.service#L17)），compose 里也是 web/worker/migrate 三角色
  （[docker/compose.yaml:44-93](docker/compose.yaml#L44-L93)），并明确靠 WAL 支持多进程（[worker.service:6](deploy/dsh-review-worker.service#L6)）。
  也就是说“并发”早就存在，且已由 WAL + `busy_timeout=5000`（[db.ts:56](src/server/platform/db.ts#L56)）处理。
  把池 `max` 提到 20，等于**在单进程内人为制造 20 个独立写者**，直接加剧 `SQLITE_BUSY`；
  `synchronous = FULL`（[db.ts:54](src/server/platform/db.ts#L54)）下每个写都要 fsync，20 条连接争一把写锁只会让 busy 从“很少触发”变成“常态超时”，表现为随机 500。
- **建议**：并发改造必须先写明“读连接 vs 写连接”的分工与写锁争用预期。SQLite 的正确扩容方向是 **WAL + 单写者 + 多读连接**，不是对称池。

### P1-2 `/api/health` 加池指标会撞上本端点自己定的契约
- **哪部分**：proposal.md:29-30「暴露池状态到 /api/health」
- **后果**：该端点**刻意**只在公网 body 下发 `status / checked_at / schema_version`
  （[health.ts:4-8](src/server/platform/health.ts#L4-L8)、[route.ts:10-12](src/app/api/health/route.ts#L10-L12)），逐项细节必须 `?detail=1` + `x-health-token`（[route.ts:23-28](src/app/api/health/route.ts#L23-L28)）。
  池状态（连接数、等待队列长度）是典型运维内情，塞进无鉴权部分等于给攻击者一张容量图。
  且该端点“高频调用不该成为负担”（[health.ts:5-7](src/server/platform/health.ts#L5-L7)），而读池状态往往要拿池锁。
- **建议**：池指标走 `?detail=1` 分支或独立端点；且**不得**因池状态把 `status` 打成 fail —— `statusForHealth` 只有 fail 才 503（[health.ts:150-152](src/server/platform/health.ts#L150-L152)），编排器会因一次瞬时排队把服务重启。

### P1-3 池化会破坏在线热备与迁移的两条前提
- **哪部分**：proposal.md 全文（未提备份与迁移）
- **后果**：(a) 备份用 SQLite 在线热备 API `db.backup()`，前提是“不阻塞写入、由 SQLite 保证一致性”（[scripts/backup.mjs:6-8](scripts/backup.mjs#L6-L8)）；
  池化后哪个连接持有备份游标、备份期间该连接能否被借出改状态，方案没交代，最坏情况是**备份产物不可恢复**。
  (b) 迁移执行器要求“唯一维护进程”（[compose.yaml:4](docker/compose.yaml#L4)、[compose.yaml:26-27](docker/compose.yaml#L26-L27)），
  而 `PRAGMA foreign_keys` 在事务里是空操作（[migrate.ts:33-44](src/server/platform/migrate.ts#L33-L44)）——这就要求迁移拿到一条**独占且未被他人共享**的连接；池化会消解 `openDatabase` 的独占语义。
- **建议**：迁移与备份一律使用“独占连接”（直接 `openDatabase`），池只服务请求路径；把这条边界写进方案。

### P2-1 池参数与公式自相矛盾，无一处可验证
- **哪部分**：proposal.md:12-14 vs 25-27
- **后果**：公式算出 9 却“取 20 留余量”（proposal.md:27）—— 余量 122% 不是余量，是拍脑袋；
  `min: 5` 常驻与 `idleTimeoutMillis: 30000` 生命周期冲突未解释；无任何“怎么证明调对了”的验收标准，而仓库本有验收脚本传统（`scripts/acceptance-*.mjs`）。
- **建议**：删掉公式。容量结论只能来自实测曲线（并发数 vs p95 延迟 vs 错误率），并落一份 `docs/实测记录-*.md`。

### P2-2 预期效果无度量口径
- **哪部分**：proposal.md:32-34
- **后果**：“并发请求不再排队”“吞吐提升 3-5 倍”两条都无法证伪：没有基线、没有压测脚本、没有回归门禁。
  按仓库已验证的教训（[docs/实测记录-M9第三次-经验假设证伪.md](docs/实测记录-M9第三次-经验假设证伪.md)），未验证的收益承诺会直接变成误导。
- **建议**：把“预期”改写成可执行验收：压测工具、并发梯度、目标 p95、以及“不达标则回滚”的判据。

## 三、如果真实瓶颈确实在 DB，怎么改才对

1. **先证明**：加事件循环延迟与慢查询埋点，确认瓶颈是 DB 等待还是 CPU（Markdown/sharp）。
2. **只读分流**：`openDatabase({ readonly: true })` 已支持（[db.ts:22-23](src/server/platform/db.ts#L22-L23)），为纯查询路径开一条只读连接即可让 WAL 的“多读并发”真正生效，**不动事务路径**。
3. **减锁时间**：压短事务、重新论证 `synchronous` 取值（[db.ts:54](src/server/platform/db.ts#L54)），比加连接有效得多。
4. **异步化**：真正并发的重活（图片、Markdown、外部拉取）本来就该踢到 worker 进程（已有 `background_tasks` 与 `MAX_CONCURRENT_TASKS=4`，[tasks/constants.ts:16](src/server/modules/tasks/constants.ts#L16)），而不是在 web 线程里抢连接。

## 四、总体结论
**驳回当前方案**。理由：(1) 前提事实错误（不存在 await 排队）；(2) 目标与手段不匹配（池治不了同步驱动）；
(3) 会破坏 4 条已被测试与断言钉住的不变量（外键、延迟外键、事务上下文、在线热备）。
若作者坚持方向，须重写为“SQLite 单写者 + 只读连接分流”并补齐验收度量；否则请改回“先度量、再优化”。
