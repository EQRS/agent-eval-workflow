# M11 全文搜索方案 · SRE（运维）评审

> 评审对象：`atb/tasks/m11-fulltext-search/environment/proposal.md`（48 行）
> 评审身份：SRE / 运维
> 事实基线：`migrations/0014_review_search.sql`、`src/server/modules/search/index.ts`、`src/server/platform/{db,migrate,health,rate-limit,tx}.ts`、`src/server/workflows/publish-review.ts`、`src/app/api/plugins/search/route.ts`、`docker/compose.yaml`、`docker/web.Dockerfile`、`docker/nginx/snippets/proxy-common.conf`、`deploy/dsh-review-web.service`、`scripts/backfill-review-search.mjs`、`scripts/backup.mjs`

## 0. 结论

**不建议按现方案实施。** 从运维角度看，这份方案有三个不可接受的形态：

1. **静默失效**：没有分词器选择（中文必挂）、没有重建入口、没有健康检查项——坏了不会报错，只会返回空结果（§1/§2/§8）。
2. **没有隔离与钝化手段**：单条查询串直接拼进 SQL、无 LIMIT、无超时、无限流；而 `better-sqlite3` 是**同步**驱动，一条慢查询就能把整个 web 进程连同所有用户的请求一起卡死（§3/§4/§10）。
3. **没有变更/回滚剧本**：新迁移文件、登记表、回填脚本、回滚路径一个都没写（§5/§6/§12）。

一句话：**这份方案把仓库自己在 0014 里已经写好教训的东西，重新踩了一遍。**

---

## 1. 中文搜索在默认分词器下必挂，静默返空 —— 方案第 12–17 行 + 第 47 行

**① 问题**：`CREATE VIRTUAL TABLE plugin_search USING fts5(...)` 没有写 `tokenize`，即用 FTS5 默认的 `unicode61`；而第 47 行预期「支持中文与英文搜索」。

**② 后果**：
- 仓库已实测并存证：`migrations/0014_review_search.sql:5-9` 原文写「SQLite 的 `unicode61` 按"词"切，中文整段算一个词，**子串搜不到**（搜"记忆"匹配不了"记忆插件"）」。中文长句被当成一个 token，插件名/描述是自然语言混排，搜「记忆」「检索」这种常见两字词几乎必然 0 条。
- 最坏的是**失败形态**：接口 200、返回空数组。health 全绿、日志无异常、监控无告警。这类故障通常在上线后几天才由用户反馈暴露，而那时索引里已经是全量脏数据。
- 与方案第 47 行「支持中文」直接互斥——需求与手段不可同时成立。

**③ 建议**：
- 照 0014 的走法：`tokenize = 'trigram'`，并把它写进注释的边界一起复制过来：**trigram 只对 ≥3 字符查询生效，1–2 字必须走 LIKE 回退**。
- 检索层直接复用 `src/server/modules/search/index.ts:156-187` 的分层（≥3 字符 MATCH+bm25 / 1–2 字 LIKE / 空查询全量），`escapeLikePattern` 已在 :227 提供。
- 验收用例必须覆盖「记忆」「检索」「检索测」三类查询，钉住这个边界（参考 0014 注释里"有测试钉住"的做法）。

---

## 2. 没有任何重建/回填入口：索引脱节后**无修复手段** —— 方案第 19–31 行

**① 问题**：方案只有「新增/更新时同步 upsert」，全文没有 rebuild / backfill / 全量重建；也没有对应的运维脚本。

**② 后果**：
- 方案用的是**独立**（非 external content）FTS 表——数据只存在于虚表，`rebuild` 命令不适用（0014:11-13 已经把这条理由写死）。索引一旦与真相脱节，唯一修复方式是从业务表**重灌全部行**。方案没有这一步 = 没有修复入口。
- 脱节是**必然**而非偶发：本方案的 upsert 自己就在制造脱节（§3 非事务写）、且历史数据完全没有索引（当前已收录的插件不会自己冒出来）。参照 0014 的运维史：`scripts/backfill-review-search.mjs` 是**必须单独跑一次**的（脚本头注释 :5-6 原文：「不跑这一步会出现"老文章搜不到、新文章搜得到"这种最难解释的现象」）。
- 全量重建的**写锁风险**：库是 WAL + `synchronous = FULL` + `busy_timeout = 5000`（`src/server/platform/db.ts:53-56`）。WAL 只让**读**不阻塞；重建是一个巨大写事务，与 `deploy/dsh-review-web.service` / `deploy/dsh-review-worker.service` 里的两个常驻进程抢写锁，超过 5 s 直接 `SQLITE_BUSY`。若照 0014 的 `rebuildIndex`（`search/index.ts:121-132`，单事务全量重建）直接照搬，重建期间 web 的每一次发布/写操作都可能 500。
- 方案也没有**并发保护**：两个运维同时跑重建（或定时任务重入）会互相 DELETE，谁也灌不完。

**③ 建议**：
- 提供 `scripts/backfill-plugin-search.mjs`（对齐 `backfill-review-search.mjs`）：幂等、可重入、结束时**对比行数并 exitCode=1**（那份脚本 :21-25 就是这么自证的）。
- **分批提交**（每批 200–500 行一个小事务，批间让出），并**优先在维护窗口 / 停 worker 时跑**；给出「重建期间是否允许写入」的明确口径。
- 加**单实例锁**（`runtime_heartbeats` 或独立锁行），重入时第二个人明确报错而不是悄悄并行。
- 重建要打日志：起止时间、删除数、写入数、耗时；否则「重建跑完了但没生效」无判据。

---

## 3. 非原子 upsert：一次失败就把插件从索引里抹掉 —— 方案第 23–30 行

**① 问题**：`DELETE FROM plugin_search ...` 与 `INSERT ...` 是**两条独立语句，无事务包裹**，且没有与业务写入同一事务。

**② 后果**：
- DELETE 成功、INSERT 失败（进程被杀、磁盘满、`SQLITE_BUSY`——见 §2 的 5 s 超时）→ **该插件从索引里彻底消失**，且不会自愈，除非它下次恰好又被改一次。这正是「更新」变成「消失」的经典形态。
- 即使两条都成功，索引与业务表也不在同一事务：业务回滚了、索引已写 → **搜得到但点开 404**。仓库对这件事的纪律写得很明白：`src/server/workflows/publish-review.ts:142-145`，注释原文「索引与"公开修订指针"必须在**同一个事务**里更新……少一边就会出现"搜得到但打不开"或"发布了却搜不到"」。
- 团队对「写一半」的故障模式并非没有意识：`src/server/platform/migrate.ts:97-115` 专门处理「schema 已变、登记未写」的半成品并让下次启动响亮失败。方案却在最核心的写路径上没有事务。

**③ 建议**：把 `upsertSearchIndex` 改成 `*InTx(conn, pluginId)` 形式，由**调用方在业务事务内**调用（照抄 `reindexReviewInTx`），并提供 `tx.ts` 的 `withTransaction` 包装给非事务调用方；索引字段取自**页面实际展示的值**（`plugin_field_overrides` 生效后的值，`0015_plugin_corrections.sql:49`），否则会出现「搜到的名字和页面不一样」，运维排查时极难定位。

---

## 4. 查询接口：单条请求可打满 web 进程（同步驱动 + 无上限）—— 方案第 33–41 行

**① 问题**：`GET /api/plugins/search` 无 LIMIT、无超时、无分页、无排序字段（第 39–40 行）；`q` 直接拼进 SQL。

**② 后果**：
- `package.json:26` 依赖 `better-sqlite3@13.0.3`，且 `db.ts:6-8` 注释原文「better-sqlite3 是同步驱动，等待会阻塞所属线程」。一条热门词的 bm25 排序会扫全部命中行（bm25 必须扫全命中集才能排序），期间**整个 Node 事件循环停摆**——不是"这个请求慢"，是**全站所有请求都卡**。web 容器 HEALTHCHECK 超时 5 s、retries 3（`docker/web.Dockerfile:76-77`），卡住会被编排器判死重启，形成"一搜就重启"的正反馈。
- 目前已有的同类接口是有防护的：`src/app/api/plugins/search/route.ts:20-21` 把 q `slice(0,100)`、limit 夹到 `1..20`。**方案（同路径 `/api/plugins/search`）反而把这个护栏去掉了**——同一个 URL 上换实现、换语义、换风险面，运维会以为是在改老接口。
- 输入里含 `"`、`*`、`AND`、`-`、`(` 等 FTS5 查询语法字符时直接抛 SQL 错误 → **普通用户的正常输入就能持续制造 5xx**，把错误率基线打脏（错误率是这里唯一"会响"的信号，长期被噪音淹没等于失效）。

**③ 建议**：
- 参数绑定 + 对 FTS5 查询串做**短语包裹 + 双引号转义**（`search/index.ts:56-58` 的 `toFtsPhrase`）。
- 硬上限 `LIMIT`（对齐 `CANDIDATE_LIMIT = 500`，`search/index.ts:47`）+ 分页；q 长度截断（100 字符，对齐 `catalog/service.ts` 与既有 route 的做法）。
- 给定**规模前提**再谈「< 50 ms」（见 §9）。
- 若确需在线搜索端点，加**匿名限流**：现有 `rate-limit.ts:46-55` 的 `RateLimitAction` 全是登录用户键（`enforceRateLimit` 在 :209-213 明确拒绝匿名），搜索是匿名高频只读接口，需要独立的按 IP/全局限流，否则单机可被搜索词打穿。

---

## 5. 部署与迁移：新表没有落任何部署产物 —— 方案全篇

**① 问题**：方案只说「使用 SQLite FTS5 虚拟表」，没写迁移文件、没写登记、没写部署步骤。

**② 后果**：
- 本仓的权威结构是 `migrations/*.sql`（`src/server/platform/migrate.ts:6`），迁移由**独立的 migrate 一次性进程**执行（`docker/compose.yaml:27-42`，web/worker 等它 `service_completed_successfully`）。没有迁移文件 = 表在生产根本不存在。
- 更要紧的是**自检门禁**：`REQUIRED_TABLES`（`migrate.ts:142-237`）是 health 的判据，`src/server/platform/health.ts:76-82` 中 `db.required_tables` 缺表即 `status = 'fail'` → **503 → web 容器 unhealthy → 编排不把流量放进来**。把 `plugin_search` 加进去是必须的；反过来，若迁移失败而登记没写，就会得到一个"永远 503"的库（对比 0014 的做法：`migrate.ts:175-176` 已显式登记 `review_search`）。
- 迁移文件命名有坑：`migrate.ts:51-55` 用 `localeCompare(b, 'en')` 排序。现有最后一个是 `0042`，若新建 `0043_` 正常；但**不要**写成 `043_` / `43_`，那会排到 0042 之后的位置不确定 / 或在 `0050` 之后乱序。

**③ 建议**：
- 明确交付物清单：`migrations/0043_plugin_search.sql`（`CREATE VIRTUAL TABLE` + 写在头注释里的取舍说明）、`migrate.ts` 的 `REQUIRED_TABLES` 追加、回填脚本、compose/部署文档的更新条目。
- 迁移里**不要**做数据回填（会把 migrate 的时间拉长、失败面变大）；回填单独一次跑，参照 0014 的分工。
- 写好**回滚口径**：`docs/docker约定.md` §6 明说「迁移是向前的……多出来的表/列无害」——`DROP TABLE plugin_search` 是可行的回滚动作，要写进变更记录。

---

## 6. 备份/恢复：新索引表的恢复路径未确认 —— 方案自增了库体积

**① 问题**：方案没提索引表的备份影响。

**② 后果（实测已知的坑，不是推测）**：
- `scripts/backup.mjs:78-88` 的**行数清单是硬编码白名单**，且只在异常时记 `null`；新表不在清单里 → 备份**不会**对它的副本比对，`countsMatch` 语料里也没有它。备份清单（探照灯）对新索引是盲区。
- `docs/docker约定.md:163` 的手工备份写法是宿主 `sqlite3 ... ".backup"`，但**部署里根本没有 sqlite3 客户端**：base 镜像是 `node:24-bookworm-slim`（`docker/web.Dockerfile:14`），只装了 `tini ca-certificates curl`（:19-21）；nginx 是 alpine 也没装。`deploy/dsh-review-backup.service:7` 甚至明说「宿主上不一定装得上」。照文档操作会直接 `command not found`。
- 备份保留期 14 天，索引与业务表同库 → 每份恢复点都携带一份 trigram 索引副本，磁盘余量被索引膨胀吃掉（见 §9），而 `health.ts:94` 的 `disk.free` 阈值是 200 MB——恢复点数目 × 索引体积是这个阈值的主要消耗者。

**③ 建议**：把新表加入 `backup.mjs` 的计数清单（或改为从 `schema_migrations` 动态推导）；把 `docker约定.md:160-165` 的备份写法改成 `docker compose run --rm web node scripts/backup.mjs`；给出一次恢复演练（恢复后跑回填脚本或验证索引行数）。

---

## 7. 运维可观测性：健康检查覆盖不到"索引为空"，且新增 check 有公网泄露面 —— 方案第 46–48 行

**① 问题**：方案对监控零描述；`/api/health` 现有检查只能回答"表在不在"，回答不了"索引有没有内容/是否新鲜"。

**② 后果**：
- FTS 索引为空时：`db.required_tables` ok、`db.readable` ok、`db.journal_mode` ok → 全绿，而搜索全空。这是**最恶劣的一类故障**：无告警、无日志、无 5xx，只有用户投诉。
- `health.ts:6-7` 明确写了「这个端点可能是公网可达的，业务数字不该顺手泄露」。若按常见做法直接把「索引行数 / 业务行数」塞进 `checks[].detail`，等于把业务量下发到公网（除非走 `?detail=1` + `x-health-token`，见 `docs/docker约定.md:176`）。
- 单机部署（`deploy/*.service`），没有 Prometheus/APM；能拿到的信号只有 journal 与 `/api/health`。所以**指标必须落在 health/journal 这两个面**，不能假设"有监控平台"。

**③ 建议**（照 `SearchService` 的 `rebuildIndex` 已有同类做法，成本很低）：
1. `fts.plugin_search_rows`：索引行数 / 业务可见插件数的比例，低于阈值判 `degraded`（`health.ts:149-151` 已支持 degraded 不影响 200，不会引起误重启）。
2. `fts.plugin_search_lag`：最近一次索引写入时间 vs 业务表 `updated_at` 最大值。
3. 搜索接口按 `request_id` 打一行结构化日志（`src/server/platform/log.ts`）：`q_len / hits / duration_ms`（**不要**打原始 q，避免把用户搜索词写进日志——这既是隐私也是容量问题）。
4. **不加行数到公网 detail**：只在 `checks[].ok` 上体现，数值细节走 token 保护的 `?detail=1`。

---

## 8. 容量与 SLO：「< 50ms」无前提、trigram 索引膨胀未估算 —— 方案第 46–48 行

**① 问题**：第 48 行「响应时间 < 50 ms」没有规模前提（插件数、平均描述长度、并发、冷/热缓存）；第 1 节的 trigram 方案（若采纳）按每个字符位置生成 3-gram，索引体积可达原文数倍，方案未做任何容量估算。

**② 后果**：
- 无前提的指标**不可验证、不可回归**；上线后每条都"不达标"或每条都"达标"，取决于你怎么解释它。
- `docker/compose.yaml:64-67` 数据目录与 `/app/backups` 是宿主 `/root` 下同一块盘；索引膨胀 + 14 份备份副本 + 媒体，一起挤 `health.ts:94` 的 200 MB 余量阈值。挤压到阈值以下 → health `fail` → 503 → 编排认为 web 坏了 → 重启循环（重启并不能腾出磁盘）。
- `.data` 挂载卷上 WAL 与索引文件的增长没有上限保护，方案没提清理/回收策略。

**③ 建议**：把指标写成可验收的口径——「N 个插件、描述均长 M 字符、并发 C 下 p95 < 50 ms」，并**附一次实测记录**（回填后测一次，数据量级写进方案）。容量上给一次「索引表体积 / 业务表体积」的实测比，作为磁盘余量输入。可考虑把 `fts.*` 指标写进 `scripts/diagnose.mjs` 的口径（它直连库、只有运维能跑，天然适合承载业务数字，`health.ts:8` 已经这样分工）。

---

## 9. bm25 的排序前提与"一行一插件"的退化 —— 方案第 44 行

**① 问题**：第 44 行说「按 FTS5 的 bm25 分数排序」，但表结构是 `plugin_id / name / description` —— **每个插件恰好一行**。

**② 后果**：
- bm25 的相关度是在**同一个词的多篇文档之间**比较。一篇文档就一行时，排序仍在，但列权重的意义被压缩成"命中在 name 还是 description"；而**插件名本来就几乎都是唯一命中**，实际效果更接近"命中列加权"，而不是"相关度"。
- 更实际的问题：**没有列权重、没有次级排序键**（bm25 相等时没有 tie-breaker），结果顺序在不同 SQLite 版本/不同缓存状态下可能不稳定——对用户表现为"同样的搜索，两次结果顺序不同"，对运维表现为"无法复现的排序问题"。
- 索引里存的是 `name/description` 副本；若不做 §3 的"取自页面展示值"，管理员采纳的修正（`plugin_field_overrides`）不进索引 → 搜到的名字与页面不一致，且 bm25 打分基于**错的文本**。

**③ 建议**：显式给列权重（`bm25(plugin_search, 0.0, 8.0, 1.0)` 之类，参考 `search/index.ts:53` 的取舍）+ 稳定的 tie-breaker（`ORDER BY score ASC, plugin_id ASC`）。并把"为什么一行一插件时 bm25 近似于列加权"写进方案，免得使用者对"相关度"有错误预期。

---

## 10. 无端到端限流/隔离：匿名可打的高频只读接口 —— 方案第 33–41 行

**① 问题**：`/api/plugins/search` 无任何频控；nginx 侧也没有对搜索的 `limit_req`。

**② 后果**：
- 现有 `rate-limit.ts` 的动作枚举（:46-55）与 `enforceRateLimit` 的语义（:209-213 拒绝匿名）决定了它**不能直接用于本接口**。于是这个接口成了全站唯一"匿名 + 可触发全表扫描 + 同步驱动"的组合。
- `docker/nginx/snippets/site-locations.conf:39-44` 的 `location /` 没有任何 `limit_req`，全部透传到 Next。单 IP 刷搜索词即可把 web 的 CPU 打满（同步查询串行执行，天然没有并发削峰）。
- 用户输入含 FTS5 语法字符 → 500（§4）。**失败但已消耗 CPU** 的请求同样算在容量里。

**③ 建议**：加一个"只读匿名"的限流键（IP 维度，`src/server/platform/client-ip.mjs` 已提供真实 IP 解析，compose 已设 `DSH_REVIEW_TRUST_PROXY=1`），或在 nginx 侧对目标 location 加 `limit_req`（注意：`add_header` 被 `proxy-common.conf:9-11` 的安全头依赖，新增 location 必须显式 include 该片段，否则安全头会**悄悄消失**——这是 `site-locations.conf:1-4` 已经写明过的坑）。同时在方案里给出"单实例 QPS 预算"。

---

## 11. 与既有实现的重复：这个能力已经存在且已上线 —— 方案第 3–4 行

**① 问题**：方案背景说"用户需要按关键词搜索插件名称和描述"，并新建 `GET /api/plugins/search`。但该路径**已经存在**（`src/app/api/plugins/search/route.ts`），底层是 `catalog.listPlugins` + `buildDirectoryWhere`（`src/server/modules/catalog/service.ts:272-283`），已支持按 `display_name / full_name / description` 的 LIKE 检索并带相关度排序（:799-810）。

**② 后果**：同一 URL 上要换成 FTS 实现、换返回结构（现有返回 `pluginId/displayName/pluginPath/repositoryFullName/ownerAvatarUrl`，:27-33）、换安全假设（现有限长限条数，方案没有）。**运维最怕的就是"路径不变、语义变了"**：监控面板、错误率基线、nginx 缓存与限流规则、前端调用方全部按旧口径配置，出了问题时没人会想到"其实是另一套实现"。
另外，这是典型的"问题已经解决过"：LIKE 子串方案对中文天然有效（不受分词器限制），在当前插件量级下大概率就是够用的；引入 FTS 是在换一个更大的故障面。

**③ 建议**：方案里必须写清**为什么不复用**现有目录检索（`buildDirectoryWhere`），以及若确实要上 FTS：(a) 用**新路径**（如 `/api/search/plugins`）而不是覆盖老路径；(b) 保留现有的 q 截断与 limit 上限；(c) 给出迁移期双跑与对照口径。

---

## 12. 变更与发布剧本缺失（SRE 的验收门槛）—— 方案全篇

**① 问题**：方案没有任何发布/回滚/对账步骤。

**② 后果**：本仓的发布是「`migrate` 一次性进程 → web/worker 等它成功」（`docker/compose.yaml:44-48,78-83`），备份是每日 03:00 的 `dsh-review-backup.timer`，恢复点是按需的。方案没说明：重建脚本在发布序列的哪一步跑、失败回滚到哪个版本、备份与重建的先后关系（**重建前必须先有恢复点**）。

**③ 建议**（最小剧本，直接写进方案第 5 节）：
1. 打一次 `backup.mjs --label pre-deploy`（`scripts/backup.mjs:16` 已支持该 label）；
2. `docker compose up -d` → 确认 `migrate` Exited(0)、`/api/health` 含 `plugin_search`；
3. 跑回填脚本，记录「写入 N 条 / 耗时 / 索引体积」；
4. 打点验证：「记忆」「检索」「检索测」三类查询 + 一条不存在的词，人工确认命中与延迟；
5. 回滚口径：代码回退 + `DROP TABLE plugin_search`（向前迁移的已知边界见 `docs/docker约定.md:168`）。

---

## 13. 必须补齐的最小清单（按实施顺序）

| # | 项 | 验收方式 |
|---|---|---|
| 1 | 定分词：trigram + 1–2 字 LIKE 回退 | 「记忆」「检索」「检索测」三类查询各一条用例 |
| 2 | 写路径包事务，索引取自「页面展示值」 | 注入 INSERT 失败，断言该插件仍在索引中 |
| 3 | 幂等回填脚本 + 分批提交 + 单实例锁 | 连跑两次结果一致；中断后可重跑收敛；并发时报错 |
| 4 | 查询侧参数绑定 + 短语转义 + LIMIT + 分页 | 输入 `"`/`*`/`AND` 返回 200 而非 500；热门词有硬上限 |
| 5 | 迁移文件 + `REQUIRED_TABLES` 登记 | `/api/health` 的 `db.required_tables` 含 `plugin_search` |
| 6 | `fts.*` 指标接入 health（degraded 不判 503），数值不进公网 detail | 清空索引 → health 变 degraded |
| 7 | 匿名限流 / nginx limit_req | 单 IP 高频请求被 429 而非打满 CPU |
| 8 | 容量记录：索引表体积比 + 实测 p95 | 回填后记录一次绝对值 |
| 9 | 备份清单加入新表 + 恢复演练 | 恢复后索引行数与业务行数一致 |

## 14. 一句话给作者

方案的每一节都能在仓库里找到一份**已经写好、已经踩过、已经留下注释**的对照实现：`migrations/0014_review_search.sql`（分词取舍）、`src/server/workflows/publish-review.ts:142-145`（同事务写索引）、`search/index.ts:121-132`（全量重建）+ `scripts/backfill-review-search.mjs`（幂等回填）、`health.ts:56-147`（可扩展的 health 检查）。**把这份方案改成"照 0014 再写一遍插件版"，工作量远小于重新设计，而且能直接继承它的运维经验。**
