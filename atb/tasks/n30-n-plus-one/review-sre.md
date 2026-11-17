# SRE（运维）评审：n30「列表页分类查询优化方案」

评审对象：`atb/tasks/n30-n-plus-one/environment/proposal.md`（proposal.md 第 1–35 行）
视角：**只从运维角度**（部署/回滚、容量、可观测、变更风险、数据安全）
结论：**不建议按现方案实施**。方案要改的东西在本仓库不存在（前提不成立）；
"一次 IN 批量查"这种正确形态**已经在生产运行**（`categoriesByRepository`）。真正的运维风险不是 N+1，
而是**不可观测的查询成本 + 无降级门禁 + 无回滚预案 + 备份/诊断链对它的盲区**。

---

## 一、阻断级（BLOCKER）

### SRE-1｜方案修的对象在仓里不存在，"21 次→2 次"这个指标运维侧无法采集、无法验收

① **哪部分**：方案 §1「优化前」`db.prepare('SELECT * FROM plugins LIMIT 20')` + 循环
`SELECT * FROM plugin_categories WHERE repository_id = ?`（proposal.md:13-18）；§预期「查询从 21 次降到 2 次」（proposal.md:34）。

② **后果**
- 列表页唯一数据入口是 `catalog.listPlugins`（[page.tsx:175](src/app/plugins/page.tsx#L175) → [service.ts:784](src/server/modules/catalog/service.ts#L784)），它是一条带 4 个相关子查询的 SQL（[service.ts:137-194](src/server/modules/catalog/service.ts#L137-L194)）。
- 分类取数早已是**一次 IN 批量查**：`categoriesByRepository`（[service.ts:248-269](src/server/modules/catalog/service.ts#L248-L269)；调用点 [service.ts:858-861](src/server/modules/catalog/service.ts#L858-L861)；详情页 [service.ts:760](src/server/modules/catalog/service.ts#L760) 复用），空数组有早退保护（[service.ts:250](src/server/modules/catalog/service.ts#L250)）。
- 运维后果：方案给出的**唯一验收指标无法在监控侧采集**——没有对应计数器，访问日志只有整请求 `duration_ms`（[server.mjs:62-82](server.mjs#L62-L82)），`better-sqlite3` 不出逐语句日志。上线后**无法做前后对比**，等于一次无法验证收益的发布。

③ **建议**：方案先改前提，把优化对象写成"4 个相关子查询 + `listCategories` 聚合"，并给出运维可采集的验收口径（见 SRE-3）；若真要动 `categoriesByRepository`，回归基线必须包含 [plugin-categories.test.ts:100-119](tests/plugin-categories.test.ts#L100-L119)（同仓多插件共享分类、展示名不重复）。

---

### SRE-2｜零索引支撑：要"显著变快"，方案里却没有任何 DDL / 迁移 / EXPLAIN 证据

① **哪部分**：方案 §2 只改 SQL 形态（proposal.md:24-31），无新索引、无迁移号、无执行计划。

② **后果**
- `plugin_categories` 只有主键索引（[0010_plugin_categories.sql:19-29](migrations/0010_plugin_categories.sql#L19-L29)），关联口径走 `json_each(c.topics)` —— SQLite 的 JSON 虚表**建不了索引**，每次都要展开全部可见分类的 topics。当前只有 10 行种子（[0010_plugin_categories.sql:34-44](migrations/0010_plugin_categories.sql#L34-L44)、[0011_plugin_categories_expand.sql:17-35](migrations/0011_plugin_categories_expand.sql#L17-L35)），所以今天"慢也慢不到哪去"；但分类**是数据不是代码**（[0010_plugin_categories.sql:13](migrations/0010_plugin_categories.sql#L13)，后台随时可加、不用发版），词表侧成本随分类数线性上涨，而**这条路径上没有任何监控能看见它变慢**。
- `listCategories`（[service.ts:964-1022](src/server/modules/catalog/service.ts#L964-L1022)）是 `plugin_categories × json_each × repositories × json_each × plugins` 全展开，**零缓存**，且跑在**每个**页面请求里：目录页 [page.tsx:188](src/app/plugins/page.tsx#L188)、评测页 [page.tsx:178](src/app/reviews/page.tsx#L178)、首页 [page.tsx:81](src/app/page.tsx#L81) —— 都是**匿名可达**的公开 GET。分类表多 100 行，这三条热门路径一起变慢。
- "显著变快"是没有依据的承诺，运维无法据此做容量预留。

③ **建议**：① 方案必附**改造前** `EXPLAIN QUERY PLAN`（`listPlugins`+`listCategories`）与 p95/p99 基线，改造后同口径复测；② 分类若要走量，先把 `plugin_categories.topics`（JSON 文本）物化成 `plugin_category_topics(slug, topic)` 普通表 + 索引（语义不变，只换存储形态）；③ 热路径不要每请求现算聚合，要么短 TTL 缓存（配 SRE-6 失效点），要么后台预聚合。

---

## 二、高（HIGH）

### SRE-3｜优化目标错位：真正的可观测点不是 SQL 条数，而是慢请求日志里的 `/plugins`

① **哪部分**：方案 §预期「列表页显著变快」（proposal.md:35）。

② **后果**
- 仓库已经把"慢"定义清楚了：`DSH_REVIEW_SLOW_REQUEST_MS` 默认 1000ms（[server.mjs:31](server.mjs#L31)），命中打 `slow:true` 并升 warn（[server.mjs:66-67](server.mjs#L66-L67)）；`/plugins` **不在静音名单**（[server.mjs:35-37](server.mjs#L35-L37) 只静音 `/_next/`、`/api/health`、`/favicon.ico`），所以每行都在。
- 也就是说，这次优化的唯一运维验收证据是"上线前后 `path=/plugins` 的 slow 行数与 `duration_ms` 分布"——方案对此零定义。更糟：1000ms 阈值对 SQLite 场景**过宽**，一次 300ms 的退化会被完全吞掉。
- 页面与 `/api/plugins`（[route.ts:24-47](src/app/api/plugins/route.ts#L24-L47)）走同一服务函数，页面耗时里混着 RSC 渲染，直接对比会误判收益来源。

③ **建议**：验收指标写成 `/plugins` 与 `/api/plugins` 的 p50/p95/p99 `duration_ms` + `slow` 计数（5 分钟窗口），并写明"降多少算成功"；发布前把 `DSH_REVIEW_SLOW_REQUEST_MS` 临时调到 200–300ms 采基线，发布后调回（或为目录类路径单独设阈值）。

---

### SRE-4｜优化把并发压力压到唯一数据库文件上，而写侧正在抢同一个锁

① **哪部分**：方案把 N 次往返压成 1 次更重的查询（proposal.md:24-31），未评估并发与锁。

② **后果**
- 本仓是**单文件 SQLite + WAL + `busy_timeout=5000`**（[db.ts:52-56](src/server/platform/db.ts#L52-L56)），web 与 worker 两进程共用同一库（[compose.yaml:65](docker/compose.yaml#L65)、[compose.yaml:92](docker/compose.yaml#L92)；[dsh-review-worker.service:6-7](deploy/dsh-review-worker.service#L6-L7) 注释明确"与 Web 用同一个库文件"）。
- `better-sqlite3` 是**同步驱动，会阻塞事件循环**（[db.ts:7-8](src/server/platform/db.ts#L7-L8) 自己写明）。查询越重、阻塞越久：并发上来时不是"后端慢一点"，而是**整个 Node 进程一起停住**——`/api/health` 跟着超时，编排器判 fail 重启 web（[web.Dockerfile:76-77](docker/web.Dockerfile#L76-L77) 的 HEALTHCHECK 连续失败），形成"变慢→探针超时→重启→冷启动更慢"的放大环。
- 反向看：把 N 次小查询合成一次大查询**减少了**让出事件循环的机会点，对单线程 Node 可能是负收益。方案没有讨论 HTTP 层缓存/合并。

③ **建议**：方案需给出并发假设（单机 QPS、目录页 PV 占比）与隔离手段：① 给 `listPlugins`/`listCategories` 加**短 TTL 内存缓存**（10–30s，键 = 筛选条件集合），把公开读压力从 SQLite 摘掉；② 或让目录页可静态/半静态渲染（当前 [page.tsx:22](src/app/plugins/page.tsx#L22) `force-dynamic` 是每请求必查）。上线后盯 `/api/health` 响应时间与容器重启计数——这是"事件循环被卡"最直接的信号。

---

### SRE-5｜改的是"没有降级开关"的热路径，回滚只能靠反部署整个镜像

① **哪部分**：方案全文（proposal.md 1-35）没有任何灰度/开关/回滚设计。

② **后果**
- 影响面是**匿名可达的公开首页/目录/评测三条路径**（SRE-2 列出的三个调用点），无按比例分流机制；`compose` 里 web 是单副本（[compose.yaml:44-68](docker/compose.yaml#L44-L68) 未设 replicas/scale），nginx 上游只有一个 `web:3000`（[default.conf:14-22](docker/nginx/conf.d/default.conf#L14-L22)）。**没有第二副本可以先打**。
- 回滚手段只有"重新 up 旧镜像"；若采纳 SRE-2 的物化表建议，就引入迁移，而迁移器**只有前进没有 down**（[migrate.ts:68-128](src/server/platform/migrate.ts#L68-L128)）——回滚代码 ≠ 回滚结构。
- 现方案不改结构，回滚本应很便宜——**但方案也没写这一句**，运维在变更评审时无法据此签字。

③ **建议**：补三行运维契约——① **无迁移**（若需迁移，必须附 downgrade，或证明旧码可在新结构上跑）；② 有**运行时开关**（如 `DSH_REVIEW_CATALOG_QUERY_V2=0` 回退旧实现，改环境变量重启即生效，不发版）；③ 写清回滚动作与目标耗时（如"5 分钟内回到上一镜像"）。上线窗口避开后台导入/自动发现任务时段（`dsh-review-worker.service` 与 discovery-drip 定时器在跑写任务）。

---

### SRE-6｜分类是"运营随时改的数据"，缓存/聚合一旦落后，运维既发现不了也修不掉

① **哪部分**：方案"再按 repository_id 分组"（proposal.md:30）之后没说结果放哪、活多久。

② **后果**
- 分类写入入口是后台 `upsertCategory`/`deleteCategory`（[service.ts:1065-1147](src/server/modules/catalog/service.ts#L1065-L1147) 起），**管理员点一下、线上展示立刻变**（`is_visible` 直接决定 chip 与筛选命中，[service.ts:964-1022](src/server/modules/catalog/service.ts#L964-L1022)）。
- 一旦加缓存或做"预分组/预聚合"，就出现**缓存与分类变更不一致**：运营改了看不到变化 → 报障"改了没生效" → 值班只能重启 web 兜底（**没有失效接口，也没有命中率指标**）。这是运维最怕的一类故障：**症状是业务错，责任在运维，工具却没有**。
- 隐藏分类还有静默面：`is_visible=0` 时该插件自动落进「未分类」（[service.ts:295-312](src/server/modules/catalog/service.ts#L295-L312)），计数变化在审计之外没有任何信号。

③ **建议**：若引入任何缓存/预聚合，必须同批交付——① 分类写入成功后**主动失效**（[service.ts:1080-1122](src/server/modules/catalog/service.ts#L1080-L1122)）；② 至少一个指标（命中/回源次数、最后刷新时间）出现在 `/api/health?detail=1&worker=1` 的 `checks` 里（[health.ts:56-147](src/server/platform/health.ts#L56-L147) 已有 append 通道）；③ 值班处置路径写进 `docs/`："改了分类没生效先看什么、怎么强制刷新"。

---

## 三、中（MEDIUM）

### SRE-7｜新查询若绕过 `buildDirectoryWhere`，`total` 与 `items` 口径分叉——而这条一致性目前没有门禁

① **哪部分**：方案 §2 独立构造 `WHERE repository_id IN (...)`（proposal.md:27-29），与既有"计数与取列表共用同一 WHERE"的纪律（[service.ts:271-326](src/server/modules/catalog/service.ts#L271-L326)，`buildDirectoryWhere` 在 [service.ts:786](src/server/modules/catalog/service.ts#L786) 与 [service.ts:828](src/server/modules/catalog/service.ts#L828) 两处共用）不是同一条路径。

② **后果**：分叉后会出"总数 37、翻到第 4 页是空的"这类**自相矛盾**（`total_pages` 来自计数 SQL，[result.ts:57-67](src/packages/common/result.ts#L57-L67)）；运维侧会当成"分页 bug"报上来，排查要读两层 SQL。而 [catalog-browse.test.ts:1](tests/catalog-browse.test.ts#L1) 只钉可见性口径，**没有钉"计数与列表同源"**。

③ **建议**：方案写明分类取数改造**不得**引入第二条 WHERE 构造路径；若必须引入，加一条断言测试（同一筛选下 `COUNT(*)` 与分页 items 并集行数一致）。

### SRE-8｜诊断/备份链对"查询形态改动"是盲区：改坏了只能人眼比对

① **哪部分**：方案没有把改动纳入既有运维流程。

② **后果**：既有工具能回答"库结构对不对、worker 活没活、任务有没有积压"（[diagnose.mjs:1-49](scripts/diagnose.mjs#L1-L49)），但**回答不了"目录页为什么慢/分类为什么不对"**；`/api/health` 的 `checks` 只覆盖 db/disk/worker（[health.ts:61-137](src/server/platform/health.ts#L61-L137)），没有查询耗时，也没有分类聚合一致性。变更后若出现"分类错但接口全 200"，值班只能手敲 SQL 比对，恢复只能回滚整库备份（[compose.yaml:61-62](docker/compose.yaml#L61-L62) 的 `BACKUP_ROOT`/`BACKUP_KEEP=14`），**代价远超一次 SQL 回退**。

③ **建议**：变更单挂两条核对项——① 上线前 `node scripts/diagnose.mjs --json` 存基线，上线后同口径比对；② 在 `/api/health?detail=1` 增加一项"目录页抽样查询耗时"（一次 `listPlugins(page=1,size=12)` 的墙钟时间，超阈值判 degraded），让"慢"在健康面板可见，而不是靠用户报障。

### SRE-9｜方案给出的 SQL 形态本身有边界缺陷，直接上线会变成 5xx 尖峰

① **哪部分**：`ids.map(() => '?').join(',')`（proposal.md:28）——空列表生成 `IN ()`。

② **后果**：SQLite 对 `IN ()` 报语法错误 → 走 error 通道返回 5xx（[route.ts:44-46](src/app/api/plugins/route.ts#L44-L46)），日志里是 `status=5xx` 的 warn 行（[server.mjs:67](server.mjs#L67)），直接触发值班告警；且**参数个数无上限**：`/api/plugins` 的 `page_size` 上限 50（[route.ts:34](src/app/api/plugins/route.ts#L34)），但 `listPlugins` 被页面、`/api/plugins/search`（[route.ts:24](src/app/api/plugins/search/route.ts#L24)，limit≤20）、草稿选插件页共用，任一调大就撞 `SQLITE_MAX_VARIABLE_NUMBER`。

③ **建议**：沿用 [service.ts:250](src/server/modules/catalog/service.ts#L250) 与 [index.ts:620-623](src/server/modules/featured/index.ts#L620-L623) 的空数组早退 + `new Set` 去重 + >500 id 分块三条，写进方案，**不要新写一份**。

---

## 四、低（LOW）/ 备注

### SRE-10｜"21 次"对不上本仓的取数方式
① 方案 §背景（proposal.md:5）与 §预期（proposal.md:34）。② 列表页每页 12 条（[page.tsx:24](src/app/plugins/page.tsx#L24)），每请求固定三查：计数 + 明细 + `categoriesByRepository`，外加页面级 `listCategories`（[page.tsx:188](src/app/plugins/page.tsx#L188)）与 `featuredCountsForPlugins`（[page.tsx:190](src/app/plugins/page.tsx#L190)），空结果再多一次计数（[page.tsx:194](src/app/plugins/page.tsx#L194)）。**任何口径都不是 21**，拿它当容量评审输入会让评估偏。③ 方案里的数字必须标注来源（EXPLAIN/日志/压测），不标来源的不进评审结论。

### SRE-11｜部署面事实（供变更单使用，非方案缺陷）
- 容器：web 单副本、不发布端口，只经 nginx（[compose.yaml:44-68](docker/compose.yaml#L44-L68)），`restart: unless-stopped`、`stop_grace_period: 15s`。
- 裸机：`WorkingDirectory=/srv/dsh-plugin-review`（[dsh-review-web.service:11](deploy/dsh-review-web.service#L11)），库路径来自 `.env.local`；迁移目录按 `process.cwd()/migrations` 解析（[migrate.ts:75](src/server/platform/migrate.ts#L75)）——改工作目录 = 找不到迁移。
- 备份：`BACKUP_ROOT` + `BACKUP_KEEP=14`（[compose.yaml:61-62](docker/compose.yaml#L61-L62)）；任何含结构变更的版本**上线前必须手动打一次 `pre-deploy` 备份**。

---

## 五、给方案作者的"可签字"清单（运维侧最小要求）

1. 重写前提：优化对象 = `listPlugins` 的 4 个相关子查询 / `listCategories` 聚合；删除"plugins 表 N+1"与"`plugin_categories.repository_id`"两个假设（[0010_plugin_categories.sql:19-27](migrations/0010_plugin_categories.sql#L19-L27) 无此列）。
2. 附**改造前后** `EXPLAIN QUERY PLAN` + `/plugins`、`/api/plugins` 的 p50/p95/p99 与 slow 计数基线（SRE-2、SRE-3）。
3. 明确**无迁移**，或附 downgrade / 结构向前兼容证明（SRE-5）。
4. 提供运行时回退开关与回滚耗时目标（SRE-5）。
5. 任何缓存/预聚合同时交付：写入失效点 + 健康面板指标 + 值班处置文档（SRE-6）。
6. 沿用 `categoriesByRepository` 的空数组早退与去重，不新造取数路径、不引入第二条 WHERE 构造（SRE-9、SRE-7）。
7. 变更单挂 `node scripts/diagnose.mjs --json` 的改造前后快照，以及 `tests/plugin-categories.test.ts`、`tests/catalog-browse.test.ts` 必须全绿（SRE-1、SRE-8）。
