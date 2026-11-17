# SRE（运维）评审：m11 插件全文搜索方案

评审对象：`atb/tasks/m11-fulltext-search/environment/proposal.md`（48 行）
参照实现：`src/server/modules/search/index.ts`、`migrations/0014_review_search.sql`、`src/server/platform/db.ts`、`docker/compose.yaml`、`deploy/*`、`docs/备份与恢复.md`
视角：仅 部署 / 回滚 / 可观测 / 容量 / 故障域。不评审鉴权与注入（安全组另有报告）。

---

## 结论摘要

方案把「检索」当成一个纯功能来写，**没有一行涉及上线动作、回滚路径、指标与容量**。
在现有架构下它有 4 个会直接变成线上事故的点：
① 没有迁移文件与回填脚本 → 部署即"新插件搜得到、老插件搜不到"，且无回滚；
② 由 web 进程在请求路径上同步写 FTS → 同一条写被 web/worker 两个进程争用，SQLite 单写锁放大成接口超时；
③ 全部故障都表现为「搜索框返回空」，而现有 health/diagnose 两类可观测手段都看不见索引；
④ trigram 分词会把 FTS 表体积放大约一个数量级，而磁盘只剩 200MB 就判 fail。

---

## 问题清单

### P1【阻塞】方案没有迁移文件，也没有回填/重建脚本 —— 部署不完整、回滚无路径
**① 哪里有问题**：方案 §1「搜索实现」（proposal.md:8-17）只给了 `CREATE VIRTUAL TABLE` 的片段，没有说明它作为哪一号迁移落地；§2（proposal.md:19-31）假定索引只从"新增/更新"开始积累，对**存量插件**没有任何交代。

**② 后果**：
- 仓库的迁移是带编号的顺序执行器（`src/server/platform/migrate.ts:51-55` 按文件名排序、`:88-123` 逐个在独立事务内执行并登记 `schema_migrations`）。当前最新为 `migrations/0042_audit_immutable.sql`，FTS 表必须先有 `0043_*.sql` 并登记进 `REQUIRED_TABLES`（`src/server/platform/migrate.ts:142-237`），否则 `selfCheck` 不把它当作结构的一部分，缺表也判"齐全"。
- 生产编排是 `migrate` 一次性容器跑完、web/worker 才启动（`docker/compose.yaml:27-42`、`:46-48`、`:81-83`）。没有迁移文件 = 部署后表根本不存在，接口报 SQL 错。
- 即便表建出来，存量插件在索引里是 0 行 —— 这正是仓库里已经踩过的坑，`scripts/backfill-review-search.mjs:5-6` 专门写了一段注释解释它。**同一类事故会原样复现**。
- 回滚：`docker compose stop` 把镜像退回旧 tag 后，若把库从备份恢复，`plugin_search` 会随备份一起消失，但 web 的新代码（若未同步回退）仍会 MATCH 这张不存在的表 → 搜索接口 500，且这类"代码与 schema 不一致"在现有 health 里**不会**被判 fail（`src/server/platform/health.ts:76-80` 只查 `REQUIRED_TABLES`，而 `REQUIRED_TABLES` 里没有它）。
- 备份清单只数 9 张表（`scripts/backup.mjs:78-88`），**不含任何 FTS 表**，恢复后"索引对不对"没有任何对账证据。

**③ 建议**：
1. 迁移落为 `migrations/0043_plugin_search.sql`，同时把 `plugin_search` 加进 `REQUIRED_TABLES`（`migrate.ts:142-237`）与 `scripts/diagnose.mjs:28-34` 的同名清单。
2. 配一个幂等的 `scripts/backfill-plugin-search.mjs`，照抄 `backfill-review-search.mjs` 的双重核对（重建条数 vs `COUNT(*)`，不等就 `process.exitCode = 1`）。
3. 在 `docs/备份与恢复.md` 的恢复步骤（`docs/备份与恢复.md:60-75`，第 74 行 `curl /api/health`）后追加"跑一次回填并看条数"这一步。
4. 明确回滚定义：**索引是派生数据，回滚不需要备份**——回滚后在新代码上跑回填即可；把这句话写进方案，避免运维在事故中去做库恢复这种重操作。
5. FTS 表可无损重建，建议把"重建索引"做成一条线上可执行的运维命令并对齐 `deploy/` 里 `docker compose run --rm --no-deps` 的既有形态（参考 `deploy/discovery-drip.sh:28`）。

---

### P2【阻塞】索引写在请求路径上、由 web 与 worker 两个进程争用同一把 SQLite 写锁
**① 哪里有问题**：方案 §2（proposal.md:19-31）"插件新增或更新时同步写入"，`upsertSearchIndex` 是两条独立 await 的 `DELETE` + `INSERT`，且没有说明在哪个进程、哪个连接、是否在既有事务内执行。

**② 后果**：
- 现有写入方是 web 进程（`createCatalogService` + `withTransaction`，`src/server/modules/catalog/service.ts:424-521`）；而插件收录实际由常驻 worker 经 `background_tasks` 执行（`deploy/dsh-review-worker.service:6-7` 明确写"提交了却永远不公开的第一嫌疑就是它没起"）。方案没有指定归属进程 → 极可能出现**两边各写一次**，第二方在 `busy_timeout = 5000`（`src/server/platform/db.ts:56`）耗尽后抛 `SQLITE_BUSY`。
- `better-sqlite3` 是**同步阻塞**驱动，等待期间整个 Node 线程停摆（`src/server/platform/db.ts:6-8` 的并行纪律写得很清楚）。一次 5 秒的锁等待 = 该 web 实例上**所有请求**一起卡 5 秒。这正是这条纪律要求"事务保持短小、不在事务内做外部工作"的原因，而方案把索引写放进了业务请求路径。
- `DELETE` + `INSERT` 两条语句之间不是原子的：进程在这中间崩/被杀，索引里该插件直接消失（`src/server/platform/tx.ts:42-52` 的 `withTransaction` 本可解决，方案没用）。
- 与此形成对照的是仓库已有的正确写法：评测索引在**发布工作流内、同一事务**里写入（`src/server/modules/search/index.ts:87-105`，注释 `:84-85` 明说"必须一起成功或一起回滚"）。
- 写放大：全量重收录（`scripts/discover-dsh-plugins.mjs`、滴灌任务 `deploy/discovery-drip.sh:28`）时每个插件都要重建整条 FTS 行。

**③ 建议**：
- 索引写**只在 worker 侧、与业务写同一事务**完成，照 `reindexReviewInTx` 的形态抽一个 `reindexPluginInTx(conn, pluginId)`，由 `catalog` 的既有事务在提交前调用。
- web 侧只读，不写索引；实在要在 web 侧补偿，也必须走 `withTransaction` 的同步回调，不能用 async/await。
- 收录这类批量场景用"批量事务 + 单次提交"，避免每插件一个事务。

---

### P3【阻塞】没有部署步骤、没有健康探针能反映索引状态 —— "搜索全空"是静默故障
**① 哪里有问题**：方案全篇（proposal.md:6-48）只有表结构、写入函数、路由与排序，**没有部署/启用步骤、没有指标、没有告警、没有排障入口**；§预期（proposal.md:46-48）只有"支持中英文"和"<50ms"两句无验收方式的话。

**② 后果**：
- 现有可观测面的真实覆盖：`/api/health` 只查 `foreign_keys / journal_mode / REQUIRED_TABLES / 磁盘 / worker 心跳`（`src/server/platform/health.ts:62-137`）；`scripts/diagnose.mjs:121-133` 的数表清单里没有 `plugin_search`。也就是说：**索引为空、索引滞后、索引行数远小于 plugins 行数，全部表现为 status=ok**。
- 检索是典型的"错得很安静"的功能：一次 `DELETE` 半途、一次回填没跑、一次 worker 重启丢任务，用户看到的是"搜索没结果"，没有任何信号会把这件事推到运维面前。等到用户报障时，索引已经空了几天。
- 方案 §4（proposal.md:43-44）提了 bm25 排序但没有列权重，排序错乱这类问题在线上同样无信号可知。

**③ 建议**：
1. 起一个**索引健康判据**并接进现有健康面：`COUNT(*) FROM plugin_search` 与 `SELECT COUNT(*) FROM plugins` 的比值；比值低于阈值时让 `/api/health` 返回降级或让 `diagnose.mjs` 退出码非 0（`scripts/diagnose.mjs:15` 已有"0=没故障 / 1=有故障"的约定可直接复用）。
2. 把这个比值加进备份清单的 counts（`scripts/backup.mjs:78-88`），恢复后能自动对账。
3. 打点：检索请求量、命中 0 条的比例、`SQLITE_BUSY` 次数、P95 耗时。**"命中 0 条比例突然抬升"就是这条功能的黄金信号**，它比耗时更能提前发现索引坏掉。
4. 方案 §预期 必须改成可验收的表述：给定数据集下的 P95 延迟、以及"索引滞后容忍度"（例如插件更新后多久必须可搜到）。

---

### P4【高】容量与磁盘：trigram 索引会显著放大库体积，而磁盘只剩 200MB 就判 fail
**① 哪里有问题**：方案 §1（proposal.md:11-17）建的是**普通（contentless=否）FTS5 表**，没有指定分词器；而仓库既有评测索引用的是 `tokenize = 'trigram'`（`migrations/0014_review_search.sql:18-24`）。方案对"用哪种分词、索引多大、能放多少插件"完全没有估计。

**② 后果**：
- trigram 是三字滑窗，索引体量按正文长度近似 3 倍膨胀。`migrations/0014_review_search.sql:5-9` 已实测记录 trigram 只对 ≥3 字符生效（2 字查询返回 0 条）。方案 §预期 却写着"支持中文与英文搜索"——**中文两字词（"记忆""检索"）在这种分词下搜不到**，这是刻意跳过还是没验证，必须由作者回答。
- 库是单文件 + WAL，与媒体、备份在同一个宿主盘上（`docker/compose.yaml:41`、`:65-67`）。`/api/health` 在剩余 <200MB 时直接判 `fail` 返回 503（`src/server/platform/health.ts:50`、`:90-102`），接着 nginx 等的就是 `service_healthy`（`docker/compose.yaml:115-118`）→ 编排层会把整个站点标成不健康。索引膨胀 + 每次备份还要在 `backups/`（同盘）再落一份完整库副本（`scripts/backup.mjs:59-65`、保留 14 份 `docker/compose.yaml:62`）→ 磁盘倍增风险真实存在。
- `UPDATE`/`INSERT` 到 FTS 表会写 WAL，插件描述很长时单条写入的 WAL 增量不可忽视。

**③ 建议**：
1. 明确分词器并与评测搜索统一：若用 trigram，就必须同时给出 1–2 字查询的回退方案（现成的正确做法是 `src/server/modules/search/index.ts:156-186` 的 `LIKE` 回退，且必须走 `escapeLikePattern`，`src/server/modules/catalog/service.ts:221-227` 已解释不加转义会让 `%` 匹配全表）。
2. 给出容量估算表：当前插件数、平均描述长度、trigram 膨胀系数、目标规模下的库体积与 WAL 峰值；并据此核对 `minFreeMb` 是否需要上调（`health.ts:50`）。
3. 若描述文本长且只搜名称/简介，考虑对 description 做截断后再入库（例如取前 N 字符），把膨胀控制在可预期范围。
4. 说明是否有意采用 `contentless` 模式——本场景不需要回读原文（只要 `plugin_id`），contentless 能省掉一份正文副本；但要注意 bm25 需要列内容，取舍要写进方案。

---

### P5【高】方案与既有 `/api/plugins/search` 路由冲突，等于绕开已上线的实现另起一套
**① 哪里有问题**：方案 §3（proposal.md:33-41）规定接口为 `GET /api/plugins/search?q=<关键词>`。

**② 后果**：
- 该路径**已经存在且已上线**：`src/app/api/plugins/search/route.ts:1-40`，它走的是 `catalog.listPlugins`（`src/server/modules/catalog/service.ts:784-895`），名称/仓库名/简介三路 `LIKE` 加相关度排序，并带 `limit` 上限 20（`route.ts:21`）与字段裁剪。
- 两者并存的最坏情况：不同调用方打到不同实现，同一个关键词返回不同结果集与不同字段形状 → "精选项选不到插件"这类问题极难复现；灰度期两边结果不一致也没有对账手段。
- 无灰度/开关：方案没有给"先打开给管理员、再全量"的步骤，一旦新实现有偏差，影响面直接是全站。

**③ 建议**：
- 明确是**替换**还是**并存**。若替换：给出字段契约等价性核对（`pluginId/displayName/pluginPath/repositoryFullName/ownerAvatarUrl`，`src/app/api/plugins/search/route.ts:27-33`），并保留 `limit` 上限约束。
- 走开关灰度：`site_settings` 表（`src/server/platform/migrate.ts:161`）已有配置能力，用它做 `search.engine = like|fts` 的开关，先内部/管理员启用 24h 观察 P95 与"命中 0 条"比例，再全量。方案里要写明这个开关与回滚动作。

---

### P6【中】排障与备份面缺位：索引没有可核对的"账本"
**① 哪里有问题**：方案没有任何"索引现在有多少行、最后一次重建是什么时候、与前次相比差多少"的表述。

**② 后果**：
- 运维无法回答最基本的三个问题：索引全不全、索引是不是最新的、重建跑成了没有。`diagnose.mjs` 的定位就是"一条命令看清现在到底是什么状态"（`scripts/diagnose.mjs:3-8`），而方案没有把自己的状态接进去。
- 备份清单不进 FTS 表的另一个后果：`docs/备份与恢复.md:16-18` 承诺的"逐表比对行数与源库"对索引不成立，恢复后无法判断索引是否与业务表同一代。

**③ 建议**：索引行数、最近重建时间、重建耗时与条数输出接 `diagnose.mjs --json`（供监控消费）与备份清单 counts。

---

### P7【中】"<50ms"这个 SLO 没有测量口径，且方案未考虑首次冷启/写放大后的尾延迟
**① 哪里有问题**：proposal.md:48 只写了"响应时间 < 50ms"。

**② 后果**：没有说这是什么延迟（服务端 SQL 耗时？端到端？P50 还是 P95？），无法验收也无法告警；且 FTS 首次查询要载入索引缓存，P99 与冷启后的首个请求会显著偏离平均值。当前 web 访问日志有"每请求一行含状态码与耗时"（`deploy/dsh-review-web.service:13-15`），但那是**全站**耗时，无法单独看检索路径。

**③ 建议**：改成"检索接口 P95 ≤ 50ms、P99 ≤ 150ms（服务端处理耗时，不含网络）"，并给出取数方式（按 `route_path` 过滤 /api/plugins/search 的耗时字段）。

---

## 上线前必须补的最小清单（运维视角）

| # | 动作 | 落点 |
|---|---|---|
| 1 | 迁移 `0043_plugin_search.sql` 并登记 | `migrations/`、`src/server/platform/migrate.ts:142-237` |
| 2 | 幂等回填脚本 + 条数自校验 | 仿 `scripts/backfill-review-search.mjs` |
| 3 | 索引健康比值接入健康/诊断 | `src/server/platform/health.ts:56-147`、`scripts/diagnose.mjs:121-133` |
| 4 | 备份清单加索引表 | `scripts/backup.mjs:78-88` |
| 5 | 开关灰度与回滚动作 | `site_settings`、`docs/备份与恢复.md` 附近 |
| 6 | 分词器 + 1–2 字回退 + 容量估算 | 对齐 `migrations/0014_review_search.sql`、`src/server/modules/search/index.ts:156-186` |
| 7 | 索引写归属 worker、同事务 | `src/server/modules/catalog/service.ts:424-521` 形态 |
