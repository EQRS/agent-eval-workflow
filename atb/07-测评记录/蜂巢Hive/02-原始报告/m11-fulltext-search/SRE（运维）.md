# m11-fulltext-search · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_题库全测_验证单元_0006
> 模型 deepseek-chat · 5406 字 · 29 次工具调用

---

## SRE 评审：插件全文搜索方案（proposal.md）

先说结论：**当前方案不可上线**。它要新建一张全文索引表，但没有编写迁移文件、SQL 拼串存在注入风险、索引与源数据没有原子一致性保障，而且可以证明它是本地实测**注定失败**的实现（SQLite 构建不支持 FTS5）。

按严重度排列，每条都给出位置、后果和改法。

---

### 1. 方案没有任何迁移文件 —— 在“权威结构是 migrations/*.sql”的项目里此步不可执行

**问题位置**：proposal.md:9-16 只给出 `CREATE VIRTUAL TABLE plugin_search ...` 的片段，但 `migrations/` 下 40 个文件里**没有对应迁移**（最新到 `0042_audit_immutable.sql`）。

**后果**：
- `src/server/platform/migrate.ts:137` 的 `REQUIRED_TABLES` 是启动自检的判据；不登记新表，自检会认为“齐了”，而实际表根本不存在。反过来，一旦补登记而部署漏跑迁移，`health.ts` 的 `db.required_tables` 直接判 `fail` → 503（`health.ts:statusForHealth` 只在 fail 时返回 503）；更糟的是这类“结构性故障”会让编排器反复重启 Pod，而重启**不会**建表。
- FTS5 虚表在 SQLite 里同样以 `type='table'` 出现在 `sqlite_master`（见 `migrate.ts:REQUIRED_TABLES` 里 `review_search` 的注释），所以必须走同一套登记流程。

**建议**：写成 `migrations/0043_plugin_search.sql`，并在 `migrate.ts` 的 `REQUIRED_TABLES` 里加 `plugin_search`；同时把 `review_search` 的既有约定一并复用（独立 FTS 表、`UNINDEXED` 回查键）。

---

### 2. 环境能力未核实：本仓库已在为 FTS5 **可能不存在**做兜底，新方案却当它必然可用

**问题位置**：proposal.md:9（“使用 SQLite FTS5 虚拟表”）在全文里没有任何 `sqlite_version()` / 编译选项校验。而 `migrations/0014_review_search.sql:5-7` 的记录显示，团队此前必须写清 trigram 分词的边界；`modules/search/index.ts` 的头部注释也说明检索是**混合策略**（≥3 字符走 MATCH，1–2 字符走 LIKE 回退）。

**后果**：如果目标镜像的 better-sqlite3 / 系统 SQLite 未编译 `ENABLE_FTS5`，`CREATE VIRTUAL TABLE ... USING fts5` 会在**启动迁移阶段**抛错。因为 `migrate.ts:122` 把每个文件包在一个事务里，失败会回滚——这算好的一半；坏的一半是：**进程起不来，且每次重启都同样失败**，属于“部署死锁”，只能人工介入回滚镜像。

**建议**：迁移前加一道前置自检（`SELECT sqlite_compileoption_used('ENABLE_FTS5')` 或 `PRAGMA compile_options`），失败时给出明确指引；并在部署文档/`health.ts` 里把它变成可诊断项。别让它在 CI 通过、上生产才炸。

---

### 3. 索引写入不在业务事务内，且“先 DELETE 后 INSERT”无原子性

**问题位置**：proposal.md:22-31 的 `upsertSearchIndex(plugin)` —— 两次独立 `await db.prepare(...).run(...)`，没有事务包裹，且是 async 风格。

**后果**：
- 两条语句之间进程崩溃 / 语句报错 → 插件**从搜索结果里消失**（旧行已删、新行未插）。这是“静默数据丢失”，没有任何报警会触发，只有用户报“搜不到”才会发现。
- 与 `src/server/platform/tx.ts` 的纪律直接冲突：仓库明确要求“跨模块流程创建一次短事务，参与模块共用同一受限事务上下文”。而现有实现 `modules/search/index.ts:reindexReviewInTx(conn, reviewId)` 已经把索引写入做成**同事务**入口，并注释说明“必须一起成功或一起回滚，否则会出现‘搜得到但打不开’或‘发布了却搜不到’”。新方案是这条已经踩过的坑的重演。
- `better-sqlite3` 是同步驱动（`db.ts:8-10` 注释），`await` 一个同步 `.run()` 只会让出事件循环，`withTransaction` 的同步回调也无法容纳 `await`——照抄这段代码会得到**半事务**的假象。

**建议**：提供 `upsertSearchIndexInTx(conn, plugin)` 同步版本，由调用方（收录/更新流程）在自己的 `withTransaction` 里调用；非事务场景再包一层 `withTransaction`。

---

### 4. 索引维护只覆盖“新增/更新”，没覆盖删除、改名、下架、批量导入

**问题位置**：proposal.md:20（“插件新增或更新时同步写入”）——仅此一处触发点。

**后果**：
- 插件被下架/删除 → `plugin_search` 里留孤儿行，搜索结果出现**点进去 404 的条目**。
- `plugins.display_name` / `description` 由探针刷新（`migrations/0005_plugin_directory_info.sql:44-52`，含 `info_updated_at`），且**允许为 NULL**（“拿不到就留 NULL”）。方案没规定“资料更新但未视为‘更新’”这条路径要不要重建索引 → 索引与展示字段长期不一致。
- 项目存在批量导入（`0031_plugin_import_batches.sql`、`0032_author_import_inputs.sql`）。批量导入若逐条 `await` 走本方案，N 个插件 = 2N 次独立写事务：WAL 下反复提交，导入期间锁竞争与耗时都会显著放大（`busy_timeout` 默认 5000ms，超时即失败）。

**建议**：
- 索引维护改为“在同一个业务事务里同步调用”的 3 个入口：新增/更新、删除/下架、批量导入（**单事务批量写**）。
- 提供 `rebuildIndex()` 回填脚本（现有 `review_search` 已有 `rebuildIndex()` 可参照），并把“索引行数 vs 插件数”的偏差做成运维指标。

---

### 5. 排序与查询形态会随数据量退化，而预期写的是“< 50ms”

**问题位置**：proposal.md:14（`plugin_id UNINDEXED`）+ 47（“响应时间 < 50ms”）+ 39-45（查询只取 `plugin_id`，无 LIMIT、无分页）。

**后果**：
- `SELECT plugin_id FROM plugin_search WHERE plugin_search MATCH ?` **没有 LIMIT**：命中几千行时一次性全取回落 JS，再逐行回查插件表 → 典型的 N+1，内存与延迟双爆。
- 方案没写 `ORDER BY bm25(...)`（第 4 节只在文字里说“按 bm25 排序”，SQL 里没有）——FTS5 默认按 rowid 返回，**相关度排序不会发生**，第 4 节等于没实现。
- 50ms 的承诺没有给出测量口径（数据量、P95/P99、并发数），无法验收，也无法在容量不足时触发告警。

**建议**：SQL 里显式 `ORDER BY bm25(plugin_search, ...) LIMIT ? OFFSET ?`；展示字段用一次 JOIN 取回（避免 N+1）；把预期改写为可验收指标（例如“1 万插件规模下 P95 < 50ms @ 20 QPS”），并纳入压测与监控。

---

### 6. 中文搜索的承诺与 FTS5 默认分词器自相矛盾（需运维侧确认）

**问题位置**：proposal.md:47（“支持中文与英文搜索”）vs proposal.md:11-15（未指定 `tokenize`，默认 `unicode61`）。

**后果**：`0014_review_search.sql:5-9` 已经记录过这次的实测结论——`unicode61` 让中文整段成为一个词，**子串搜不到**；所以现有的 `review_search` 用了 `tokenize='trigram'`，并且因为 trigram 对 <3 字符无效而保留 LIKE 回退（`modules/search/index.ts` 头部）。新方案用默认分词器，等于把 14 号迁移已经修正过的缺陷重新引入：用户搜“记忆”搜不到“记忆插件”。这类问题在 SRE 侧表现为**用户报障噪声与支持工单上升**，但监控指标（5xx、延迟）全绿，最难排查。

**建议**：显式 `tokenize='trigram'`，并在服务层保留 1–2 字的 `LIKE ... ESCAPE` 回退（沿用现有实现，不要新写一套）。上线前用中文子串用例（2 字、3 字、中英混排）做端到端验证。

---

### 7. `q` 直接拼进 SQL 字符串 —— 注入与稳定性双风险

**问题位置**：proposal.md:42-45：``const sql = `SELECT ... MATCH '${q}'```。

**后果**：
- 单引号注入即经典 SQL 注入；即使不注入，用户输入一个 `"` 或 `*`、`AND`、`NEAR` 也会让 FTS5 抛**语法错误** → 接口 5xx。运维视角是“搜索接口错误率随用户输入随机波动”，无法靠值班手册定位。
- 现有实现已经把这件事做对了：`modules/search/index.ts:toFtsPhrase()` 对内部双引号做转义，并且 `q` 被 `trim().slice(0, 100)` 截断。新方案应当复用而不是重写。

**建议**：改为参数绑定 + phrase 转义（`"${input.replace(/"/g, '""')}"`），并对 `q` 做长度上限（如 100 字符）。这条必须在上线前修复，属于安全与可用性的交集。

---

### 8. 备份/回滚/容量：新虚表是“可重建数据”，但方案没说明运维处置方式

**问题位置**：proposal.md 全文（无运维章节）。

**后果**：
- FTS5 虚表带**影子表**（`plugin_search_data` / `_idx` / `_content` / `_docsize` / `_config`），备份体积与恢复步骤会变化；恢复备份后索引可能落后于业务表（`restore-backup.mjs --force` 整库覆盖的风险，参见 `0042_audit_immutable.sql` 的同类讨论）。
- 回滚镜像时，新表会**留在库里**（迁移器只前进不回退，`migrate.ts` 按文件名去重）。老版本代码不知道它存在——通常无害，但必须有明确说明，否则下次有人手工 `DROP` 会连带删掉影子表。
- 磁盘水位：`health.ts` 有 200MB 默认下限，索引翻倍增长（trigram 尤其膨胀）时没有对应告警项。

**建议**：在方案里补一节“运维”：索引可随时 `rebuildIndex()` 重建（不需要进备份关键路径）、恢复后必须重建、镜像回滚时保留表、以及把索引大小与偏差纳入 `health.ts` / `diagnose` 的检查项。

---

### 优先处理顺序

1. **第 7 条（SQL 拼串）** 与 **第 3 条（无事务）** —— 安全 + 数据丢失，必须改。
2. **第 1、2 条（无迁移 / FTS5 能力）** —— 不改就根本部署不起来。
3. **第 6、4 条（中文分词 / 触发点不全）** —— 上线后必然产生用户报障。
4. **第 5、8 条（性能验收 / 运维处置）** —— 可在实现阶段补齐，但 50ms 的承诺必须换成可验收口径再对外说。