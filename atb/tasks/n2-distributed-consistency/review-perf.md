# 性能评审：settings 读取加进程内缓存

评审对象：`tasks/n2-distributed-consistency/environment/proposal.md`（40 行）
评审角度：缓存击穿、全表扫成本、内存占用
核实基线：`src/server/modules/settings/index.ts`、`src/server/modules/forum/posts.ts`、`src/server/platform/{db,rate-limit,health,migrate}.ts`、`migrations/0001_baseline.sql`、`docker/compose.yaml`

## 摘要（前 5 条最重要）

1. 【严重】无 single-flight，冷启动/过期瞬间并发全表扫 + 全量 JSON.parse 叠加在同步驱动上 — 方案:12-21（`cached/if miss → SELECT` 无并发合并）、settings/index.ts:344-352（每 miss 都 `JSON.parse` 全表并新建 Map）、db.ts:6-8（better-sqlite3 同步，等待会阻塞事件循环）— 建议：包一层 single-flight（module 级 `let inflight: Promise<Map>|null`，miss 时复用同一 Promise），命中路径零 await；把「查库」与「解析」拆开，解析结果随缓存一起存，避免每次 miss 重算 22 项 `SETTING_DEFINITIONS` 的 merge。
2. 【严重】方案所谓的"全表扫"其实不是瓶颈，真正的每请求全表扫在 `selfCheck()`，且它被容器 HEALTHCHECK 每 30s 打一次 — 方案:4-5 把成本归因到 `effectiveSettings()`，但 settings/index.ts:344 是 `site_settings` 单表 `key` 主键小查询（k=22 行，1-2 页 B-tree，微秒级）；真正的全表成本在 migrate.ts:249-263（`sqlite_master` 全扫 + 26 表名比对 + `PRAGMA foreign_key_check` 对**全库所有表**做表扫）+ 15 条 `PRAGMA`，由 web.Dockerfile:76-77 每 30s 触发一次、admin/page.tsx:159-163 每次进后台触发一次 — 建议：把 selfCheck 的 `foreign_key_check` 降级为低频（启动一次/每日一次/显式 `?deep=1`），HEALTHCHECK 走轻量接口或让 server.mjs 缓存 report 10s；并按表规模分别测 `effectiveSettings` 与 `selfCheck` 再决定优化对象。
3. 【高】TTL=60s 对"配置变更可见时延"是新增的语义退化，且与 `update()` 的"同值不算变更"判定互相打架 — 方案:16（`cachedAt < 60_000`）、方案:30（只在调用方进程清缓存）vs settings/index.ts:472-488（`current = effectiveSettings(db)`，`from === to` 则 `applied.length === 0` 直接返回、不写修订、不写库）— 后果：缓存冻结期内"改 A 成 X 再改回原值"这一对操作，后一次在缓存视角下 `from === to`，整条变更被静默吞掉（无修订、无审计），管理员以为改回来了而库里仍是 X，直到 TTL 到期才对得上。建议：`update()` 的 `current` 必须绕过缓存（`effectiveSettings(db, {fresh:true})`），并且 `update()` 成功后必须同步刷本进程缓存（用返回值 `effective` 直接替换，别置 null 让下一次请求再扫）；TTL 只作为"外部改库"的兜底，不从 0 开始。
4. 【高】缓存粒度和"`db` 参数"的设计不匹配真实调用面，且会让现有测试直接变红 — 方案:15 `getSettings(db)` 接受 `db` 参数，却在模块级变量里存结果：一旦有两个连接（只读连接见 db.ts:22-23、测试临时库见 tests/support/db.ts）就会把 A 库的配置返给 B 库 — 建议：缓存按 `SqliteDatabase` 实例分片（`WeakMap<SqliteDatabase, Entry>`，与 rate-limit.ts:157 `limiterCache` 同一范式），或干脆把缓存收进 `createSettingsService` 闭包里。
5. 【中】内存占用真实存在，量级取决于 `value` 列实际使用量，方案没有上限业务 — 方案:12-21 一次把整表 `key,value` 全量载入并 `JSON.parse` 后常驻；当前 `SETTING_DEFINITIONS` 22 项（settings/index.ts:49-272）里最大的是字符串项 `forum.banned_words`，且该表无 `CHECK (length(value) <= N)`（0001_baseline.sql:447-453 只有 `json_valid`），所以缓存体积 = 实际配置体积、无界 — 建议：定义每项上限（词表 ≤ 64KB 之类）并写进 `parseSettingValue`，读接口对超限项回落默认值；顺带消除 forum/posts.ts:237-253 的**旁路读**。

## 逐项明细

### P-1【严重】缓存击穿（stampede）：过期瞬间 N 个并发请求同时 miss，并全部落在同步驱动上

- **依据**：方案:12-21 只有 `let cached / cachedAt` 与普通 `if` 判断，无 single-flight、无锁、无"正在加载"标记；settings/index.ts:344 每次 miss 是 `prepare + all()`（better-sqlite3 的 prepare 不跨调用复用）；db.ts:6-8 明文写"better-sqlite3 是同步驱动，等待会阻塞所属线程"。
- **放大条件**：所有页面 `export const dynamic = 'force-dynamic'`（如 src/app/page.tsx:20、src/app/forum/page.tsx:17 等 50+ 处），没有 ISR/CDN 兜底；`assertForumEnabled()` 有 43 个调用点（src/app/api/_lib/context.ts:41），同一个 POST 里 `booleanSetting` + `rateLimitOverrideFor` 会各命中一次 `effectiveSettings`（src/app/api/feedback/route.ts:30 与 :48），即**单请求 2-3 次 getSettings** 是常态 —— 缓存命中时这没问题（这正是方案的价值），但**击穿时是 2-3 倍放大**。
- **建议**：single-flight（`if (inflight) return inflight`）；过期后**先返回旧值、后台刷新**（stale-while-revalidate）而不是让请求等；TTL 建议加抖动（`60_000 ± 5s`），避免多进程同时到期。

### P-2【严重】成本归因错位：`site_settings` 全表扫不是瓶颈，`selfCheck()` 才是

- **依据**：
  - 方案:4-5 说 `SELECT key, value FROM site_settings` 是"全表扫"引起开销。该表 PK 是 `key`（migrations/0001_baseline.sql:447-453），等于索引全扫 `k=22` 行，属"索引扫描"而非"表扫描"，成本量级是微秒。
  - 仓库里真正每次健康检查都跑全库扫描的是 migrate.ts:239-274：`sqlite_master` 全扫（249-251）、26 张必需表比对（252）、以及 `:262` 的 `PRAGMA foreign_key_check` —— 这个 PRAGMA 会遍历**全库每一张表**做外键校验，成本与数据量线性相关。
  - 触发频率：docker/web.Dockerfile:76-77 `HEALTHCHECK --interval=30s` 打 `/api/health`；src/app/admin/page.tsx:159-163 管理员每次进后台再跑一遍；`/api/health` 无任何缓存（src/app/api/health/route.ts:32-38）。
- **风险**：库长大后，每 30s 一次的 `foreign_key_check` 会在 web 进程主线程上产生可观测的抖动（同步驱动，见 db.ts:6-8），而方案对此一行未提；更糟的是方案会让人以为"settings 优化完了"而错过真正的热点。
- **建议**：`foreign_key_check` 从探针路径移除或降为每日一次；HEALTHCHECK 用轻量只读探针（或 server.mjs 侧缓存 report 10s 并对并发请求合并）；优化顺序改为「先量化 `effectiveSettings` 单次耗时 × 每请求次数」再决定要不要缓存。

### P-3【高】60s TTL 会与 `update()` 的幂等判定冲突，静默吞掉配置变更

- **依据**：settings/index.ts:472-488 —— `update()` 开头 `const current = effectiveSettings(db)`，命中缓存后拿到的是**可能已过期 60s 的旧快照**；随后 `if (from === to) continue`，若全部键都相等则 `return { revisionId: null, ... }`（486-488）：不写 `configuration_revisions`、不写 `site_settings`。
- **可复现场景**：管理员把 `home.latest_review_count` 从 6 改成 3（缓存已失效 → 正常写入）；60s 内又改回 6 → 缓存里仍是 6 → `from === to` → 整次操作被判"无变更"并返回成功，但库里是 3。管理员看到"保存成功"，实际配置是 3，审计里也没有这条记录。
- **建议**：`update()` 的读必须走**不缓存**路径；`update()` 成功后在 `db.transaction` 之外（或返回前）用返回值 `effectiveAfter` 原子替换缓存，而不是 `cached = null`（置 null 会把一次刷新代价推给下一个请求，正好落在击穿窗口里）。

### P-4【高】缓存以模块级变量存放，忽略了 `db` 参数，直接违反既有测试语义

- **依据**：方案:12 的 `let cached` 在模块作用域，与传入的 `db` 无关；settings/index.ts:343 `effectiveSettings(db: SqliteDatabase)` 是全仓唯一入口，业务又有只读连接的口子（db.ts:22-23）、测试全部使用"真实临时文件数据库 + 真实迁移"（tests/comments.test.ts:10 等），同一进程里出现多个库是常态而非例外。tests/roles-service.test.ts:386-389 走的是绕过服务层的 `INSERT`，tests/settings-wiring.test.ts:77-78 更是直接 `UPDATE site_settings` 模拟"手改库" —— 这两处在命中缓存后会拿到旧值，测试会红。
- **建议**：`WeakMap<SqliteDatabase, {at, map}>` 按连接分片（rate-limit.ts:157 的 `limiterCache` 就是这个范式，可直接照抄）；测试要能显式 `resetSettingsCache()`。顺带说明：`PRAGMA data_version` 只对**同一连接**能观测到其它连接的提交（同连接自己写库时不变），所以合适的组合是"连接级 data_version 探活 + 单行 `MAX(updated_at)` 兜底"，不是单纯 TTL。

### P-5【中】内存占用：当前量级很小，但方案没有设上限，且留下了两个"旁路读"

- **依据**：settings/index.ts:49-272 共 22 项定义，取值都很小；最大的是 `forum.banned_words`（字符串，逗号分隔词表，settings/index.ts:147-153）；`site_settings` 只对 `value` 约束 `json_valid`（0001_baseline.sql:449），没有长度上限。方案:12-21 把 `key,value` 全量载入并 `JSON.parse` 后常驻，体积等于**实际配置体积**而不是"定义体积"，没有封顶。
- **另有两处绕过 settings 模块直读该表，缓存改了它们也覆盖不到**：forum/posts.ts:237-241（`SELECT value FROM site_settings WHERE key = 'forum.banned_words'`，发帖 601 行与编辑 648 行各调一次，且都在 `withTransaction` 事务里）。
- **建议**：`parseSettingValue` 增加每项长度上限（如词表 64KB）并在读侧回落默认值，把内存上界写进代码而不是靠约定；把 forum/posts.ts 的两处旁路读改成走缓存/服务层的 `stringSetting()`，否则"配置缓存的可见性"会有第三个不一致来源。

### P-6【中】性能预期里的收益口径不成立

- **依据**：方案:39-40「每次页面渲染省 3-5 次查询」。按实际调用面，绝大多数页面是 1 次（`assertForumEnabled` / 单个 `booleanSetting`），只有写接口会到 2-3 次（feedback/route.ts:30 + :48）。真正该量化的是"省掉的那次查询耗时" —— 在 22 行的主键索引尺度上是微秒级，而缓存层新增的是每次命中的 `Date.now()`、Map 查表与分支。
- **建议**：把预期改成可验收的指标：`effectiveSettings` 单次 P50/P99（单位 µs）、每请求调用次数（加计数埋点）、以及**击穿时并发 miss 数**（应恒为 1）。不要用"省几次查询"当指标 —— 它掩盖了"省下的东西本来就很便宜"这件事。

### P-7【低】`cachedAt` 用 `Date.now()`，时钟回拨会让缓存永不过期或瞬间全过期

- **依据**：方案:13、16、19 用墙钟差值判断 TTL；本仓既有代码对时钟回拨是显式处理过的（heartbeat.ts:108-111「时钟回拨时不返回负数」）。
- **建议**：改用 `performance.now()` / `process.hrtime.bigint()` 这类单调时钟算间隔；回拨场景不必额外判断，单调时钟天然免疫。

### P-8【低】写路径的全表扫没在方案里出现，但比读路径贵

- **依据**：settings/index.ts:513-524 在同一个写事务里逐键 upsert，外加 settings/index.ts:494 的一次取号查询；db.ts:54 是 `synchronous = FULL`，一次管理员保存 = 一次 fsync × 1 个事务。取号查询因 `UNIQUE (revision_number)`（migrations/0001_baseline.sql:444）有隐式唯一索引，**不是**全表聚合，这点我先写了又核准、撤回该指控。
- **建议**：读路径上缓存之后，这条写路径相对占比会上升，应当在方案的"不在本次范围"里明确排除，并补一句"写路径成本 = 一个 FULL 同步事务"，别让读者以为写路径已经算过账。若将来要优化，方向是参考 `runtime_heartbeats` 式的单行覆写而不是每次追加快照。

## 附：一条与方案无关但同层级的性能观察（供排期参考）

`POST /api/admin/backups`（src/app/api/admin/backups/route.ts:110-119）在请求线程里用 `execFileSync` 跑子进程备份，`timeout: 120_000` —— 同步阻塞 web 的整个事件循环，最坏情况两分钟全站不响应（且 db.ts:6-8 说明这就是本仓已知的代价）。做成后台任务（`background_tasks` 表已经有）比做 settings 缓存更有性能收益。