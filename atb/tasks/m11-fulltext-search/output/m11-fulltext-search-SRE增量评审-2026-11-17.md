# M11 全文搜索方案 · SRE（运维）增量评审

> 评审对象：atb/tasks/m11-fulltext-search/environment/proposal.md（48 行）
> 视角：只从 SRE 出发——部署 / 回滚 / 可观测 / 容量 / 故障域
> 事实基线（逐行核过）：migrations/0014_review_search.sql、src/server/modules/search/index.ts、src/server/platform/{db,migrate,health}.ts、src/server/workflows/publish-review.ts:145、src/app/api/plugins/search/route.ts、src/server/modules/catalog/service.ts:272-283/784-832、docker/compose.yaml、docker/nginx/snippets/{proxy-common,site-locations}.conf、docker/nginx/nginx.conf、deploy/dsh-review-web.service、scripts/{backup.mjs,backfill-review-search.mjs,restore-backup.mjs}、server.mjs、docs/备份与恢复.md
> 说明：同目录已有 m11-fulltext-search-SRE评审.md。本文只写它没写或写偏的，重叠处显式标注（见 §11 对照表）。

## 0. 一句话结论

方案不是「没考虑运维」，而是「把仓库自己在 0014 里已经写死的运维结论逐条丢掉了」：
分词选型、非外表达不可 rebuild、索引与真相同事务、回填脚本、健康判据——这五件事本仓都有既成做法，方案一件也没接。
按现方案上线，最可能的形态不是「搜索不好用」，而是**一次注入请求就能把一份好备份变成不可恢复的备份**，以及**新表登记自检之后，代码回滚或止血动作会把全站打成 502**。

---

## 1. 方案与仓库既有实现同构，应「照 0014 再写一遍插件版」而非重新设计

**① 哪部分有问题**：方案第 6-17 行（技术方案全篇）没有引用仓库既有的检索实现，仿佛是新能力。

**② 后果**：
- 本仓已有完全同构的一份：migrations/0014_review_search.sql:18-24（FTS5 + tokenize='trigram'）、src/server/modules/search/index.ts（266 行：分层检索、toFtsPhrase、CANDIDATE_LIMIT、rebuildIndex）、scripts/backfill-review-search.mjs（幂等回填 + 行数自证）。
- 0014 头注释（:5-16）已把本方案的三个坑写死：unicode61 中文子串搜不到；独立（非 external content）表不能 rebuild、只能全量重灌；索引必须在发布事务内写。方案一条没接。
- 运维后果：现场出现「搜不到」时，值班面对的是两套检索（评测走 trigram、插件走 unicode61），排查手册要分叉。

**③ 建议**：以 0014 为模板产出插件版：迁移文件 + 检索模块 + 回填脚本 + 测试四件套一一对应；方案头部写明「本方案是 0014 的复制」，评审者只需 diff 差异项。

---

## 2. 缺迁移与部署产物 —— 方案第 8-17 行

**① 哪部分有问题**：只写了 CREATE VIRTUAL TABLE，没有迁移文件、没有版本号、没有部署步骤。

**② 后果**：
- 权威结构是 migrations/*.sql（src/server/platform/migrate.ts:6），迁移由一次性进程执行（docker/compose.yaml:27-42），web/worker depends_on service_completed_successfully（compose.yaml:44-48、:78-83）。没有迁移文件，表在生产根本不存在。
- 文件名排序是 localeCompare(b,'en')（migrate.ts:51-55）：现存量最后一份是 0042_audit_immutable.sql，必须写 0043_ 形式，不能写 43_。
- 既有评审 §5 已讲「没有迁移产物」；本文补一条：**迁移里不要做回填**（migrate 在启动关键路径上，失败即整站起不来），回填单独一次跑。

**③ 建议**：交付物清单 = migrations/0043_plugin_search.sql（含 tokenize 取舍注释）+ 回填脚本 + REQUIRED_TABLES 决策记录（见 §3）+ 部署手册条目。

---

## 3. 【回滚】把新表登记进 REQUIRED_TABLES，会让「代码回滚」和「止血删表」都变成全站 502 —— 方案全篇未提

**① 哪部分有问题**：方案没有权衡「新表要不要进启动自检 REQUIRED_TABLES」。

**② 后果**（本文与既有评审 §5 立场的分歧点）：
- 链路是硬性的：migrate.ts:142-237 是自检白名单 → health.ts:76-82 缺表即 status='fail' → docker/web.Dockerfile:76-77 的 HEALTHCHECK 失败 → web 永不 healthy → docker/compose.yaml:115-118 的 nginx depends_on service_healthy 等不到 → 入口不启动，nginx 自身报 502，整站不可用。
- 触发条件很现实：值班为止血临时 DROP TABLE plugin_search，新二进制下一次健康探测即 503，容器不 healthy，nginx 不再被放行；**止血动作本身把故障放大成全站 502**。反过来，从更早的恢复点恢复（不含该表）也会落到同一状态。
- 也就是说：把一个可选功能表达成启动硬依赖，等于把「搜索坏了」升级成「站点起不来」，与仓库「降级不静默、但不扩大故障域」的口径相反。

**③ 建议**：
- 优先**不进** REQUIRED_TABLES，改为在 health.ts 加一条独立的 fts.plugin_search 检查，失败判 degraded（health.ts:149-151 已支持 degraded 不影响 200 → 不触发重启、不阻断 nginx）。
- 若一定要进白名单，方案必须写明回滚剧本「回滚或止血前先 DROP TABLE 与登记对齐」，并在 docs/docker约定.md:168 的「迁移是向前的」旁加例外说明。
- 验收用例：删掉 plugin_search 表 → /api/health 返回 200 + degraded（而不是 503），搜索端点降级到 LIKE 路径（见 §7）。

---

## 4. 【故障域】非事务 upsert 只是表层；真风险是「把同步写事务拉长到 >5s」 —— 方案第 23-30 行

**① 哪部分有问题**：DELETE 与 INSERT 两条语句裸跑、且与业务写不同事务；更关键的是**索引维护挂在哪条路径上根本没写**。

**② 后果**：
- 既有评审 §3 已指出「DELETE 成功、INSERT 失败 → 插件从索引消失」。这里补两点它没说的：
  1. plugins.display_name / description 来自 GitHub 探测（migrations/0005_plugin_directory_info.sql:44-50），写入者是 worker。若把索引维护塞进 worker 的入库事务：worker 与 web 各一个 better-sqlite3 连接、单写者、busy_timeout=5000（src/server/platform/db.ts:53-56），于是**每入库一个插件就多一次同步写事务**，与 web 的发布事务抢锁；超 5 s 直接 SQLITE_BUSY → 任务失败重试 → 库躺在「永远忙」的均衡点上。
  2. db.ts:54 的 synchronous=FULL：每个写事务 commit 都 fsync。回填若照 search/index.ts:121-132 的 rebuildIndex（单事务全量），是一次巨型写事务 + 单次超大 fsync，回放日志暴涨。
- 反向的坑：若按「必须同事务」去照抄 publish-review.ts:142-145，索引维护就被绑进发布事务；一次 FTS5 写放大就能把全站最关键的发布链路拖死。搜索不该让发布让路。

**③ 建议**（与既有评审相反，这是我的主张）：
- 把检索索引定位成可修复的派生数据，不是强一致副本：写路径用主事务提交之后的独立小事务，失败只打 warn 并落一条待修复队列；再加每日对账（索引行数 vs 可见 plugins 行数、最近写入时间差）。
- 强一致只在一个地方要：界面能点开的插件必须搜得到。这由对账 + 回填脚本兜底，比「发布事务里多写一张表」便宜得多。
- 回填必须分批提交（每批 200-500 行小事务），方案要写明「回填期间是否允许写入」，并加单实例锁。

---

## 5. 【回滚 × 注入】方案第 39 行：一份被污染的备份，等于删除了一次恢复能力

**① 哪部分有问题**：第 39-40 行把 q 直接拼进 MATCH 字符串。

**② 后果**（安全评审讲注入本身，这里只讲运维后果）：
- server.mjs:50-52 只取 req.url.split('?')[0] 作访问日志的 path，**query 串根本不进访问日志**。注入串不会留在 http_request 行里；唯一留痕是 500 的 request_error（src/instrumentation-node.ts:50-63），同样不含 query。事后无法回答「谁、带什么参数把库搞坏了」。
- 攻击串最终会被 scripts/backup.mjs 原样打包：它在线热备整库（scripts/backup.mjs:62-65），并把行数与 integrity_check 写进 manifest.json（:71-94、:110-135）。而恢复侧 scripts/restore-backup.mjs 是**先验后写**（docs/备份与恢复.md:63-78）：sha256 不符拒绝、副本完整性不过拒绝。最坏形态是：良库被注入改坏 → 当晚 03:00 自动备份把它固化 → 14 天后所有恢复点都不可信，而全程没有任何信号（备份「成功」、integrity_check 也过，它只查页结构、不查内容）。
- 备份是通过 docker compose run --rm --no-deps web node scripts/backup.mjs 执行的（docs/备份与恢复.md:47、:57），拿到的一致性副本里包含全部被注入过的行。

**③ 建议**：
- 参数绑定 + 短语包裹与双引号转义（src/server/modules/search/index.ts:56-58 的 toFtsPhrase），这是唯一根治。
- 访问日志只记 q 的长度与命中数，原文不进日志（隐私 + 容量）；但 500 的 request_error 里应带 route_path（已有）+ query 的键名清单，让「谁在用怪参数」可检索。
- 备份侧加内容级对账：manifest.json 除行数外，记录关键表 max(updated_at) 与 operation_records 尾条 id，恢复演练时比对。这不解决注入，但能在「东西被改了」时给出当前完全缺失的信号。

---

## 6. 【容量】磁盘余量同时决定「健康」与「能不能写」，方案未做任何估算 —— 方案第 46-48 行

**① 哪部分有问题**：要新增一张随插件数增长的表，却没算容量、没写保留/回收策略，指标也没有前提。

**② 后果**：
- 三个信号被绑在同一个库上：db.ts:54 的 synchronous=FULL（写入即 fsync）、health.ts:88-103 的 disk.free（默认 200 MB 阈值，越线判 status='fail'）、health.ts:107-120 的 worker 心跳（每 10 s 一次写）。索引膨胀会同时推高：库体积 → WAL → 14 份备份副本体积 → disk.free 跌破 200 MB → health fail → 编排重启循环（重启腾不出空间）。这就是磁盘-健康-写入三信号耦合，一次磁盘紧张会以三种故障形态同时出现。
- 备份是 db.backup() 全量热备（scripts/backup.mjs:62-65），没有「跳过某张表」的选项 → 每份恢复点都带一份索引副本。这不是 FTS 独有，但 FTS 让「库」第一次真正变贵。
- 若采纳 trigram（见 §1），每个字符位置一个 3-gram，中文正文索引可达原文数倍；方案第 48 行只给「< 50 ms」，没有任何存量、并发、缓存前提。

**③ 建议**：
- 方案里给出三组实测数：插件数 N、display_name+description 平均长度 M、索引表体积 / plugins 表体积比（实测一次，别估算）；据此推导 14 份备份的额外磁盘占用，与 health.ts:94 的 200 MB 阈值一起判断。
- 把 disk.free 拆成两级：disk.free_warn（如 1 GB）判 degraded、disk.free 判 fail。现在「磁盘紧」和「磁盘没了」被压成同一个信号，响应方式却完全不同。
- 「< 50 ms」写成可验收口径：N 个插件、平均 M 字符、并发 C、冷缓存下的 p95，并附一次实测；否则它不是 SLO，是一句祝福。

---

## 7. 【兜底】不要在既有路径上替换实现；FTS 不可用时应回落到 LIKE —— 方案第 33-41 行

**① 哪部分有问题**：方案把检索挂在 GET /api/plugins/search，而这个路径已经存在且带护栏。

**② 后果**：
- src/app/api/plugins/search/route.ts:16-40 现役实现走 catalog.listPlugins；src/server/modules/catalog/service.ts:278-283 就是 display_name / full_name / description 的 LIKE 子串检索，对中文天然有效、不依赖分词器；service.ts:830 有 LIMIT ? OFFSET ?；route.ts:20-21 把 q 截到 100 字符、limit 夹到 1..20；tests/review-search.test.ts:167-168 还钉着「名称全等 > 前缀 > 简介包含」的相关度。
- 也就是说：方案要解决的问题，在当前插件量级下已经被解决过，而且更省、更稳。新方案在同一 URL 上换实现，会同时换掉返回结构、护栏、缓存/限流口径、错误率基线。
- 从运维角度这是故障面互换：用一个有状态的方案（索引会脏、会膨胀、会阻塞写），换掉一个无状态的方案（每次 LIKE、结果一定对，只是慢）。索引系统的经典失败模式是「搜得到其实没有」「有其实搜不到」，LIKE 永远不会给出这种答案。

**③ 建议**（按第 7 条兜底纪律）：
- 保留 catalog.listPlugins 的 LIKE 路径作为运行时兜底：新端点内部 FTS 抛错 / 超时 / 表不存在 → 自动回落 LIKE 并打 warn。这条必须写进方案，不能「实现时顺手加」。
- 若确实要上 FTS，改走新路径（如 GET /api/search/plugins），老路径一个字节不改，并给一次双跑对照（同一 q 下 top-10 重合率）。
- 方案里要明确「为什么现有 LIKE 不够」（是规模还是排序），否则这个变更没有验收起点。

---

## 8. 【可观测】健康检查与日志都回答不了「索引是否新鲜」 —— 方案第 46-48 行

**① 哪部分有问题**：方案对监控零描述。

**② 后果**：
- 现有 /api/health 的检查项（health.ts:61-120：db.readable / db.foreign_keys / db.journal_mode / db.required_tables / disk.free / worker.heartbeat）没有一条能回答「检索索引有没有内容、新不新鲜」。索引为空的形态是：一切 ok、接口 200、返回空数组、日志无异常、无 5xx。
- 宿主机是单机 systemd（deploy/dsh-review-web.service:19-21）或 compose，没有 Prometheus/APM → 信号只能落在 journal 与 /api/health 两个面，方案必须按这两个面设计。
- server.mjs:35-37 对 /api/health 与静态资源静默 → 健康检查本身不进访问日志，任何要观察的信号都必须由 health 响应体或业务日志承载。

**③ 建议**：
1. fts.plugin_search_rows：索引行数 / 可见 plugins 行数，低于 0.9 判 degraded（数值不要进公网 detail，health.ts:6-7 已明确纪律）。
2. fts.plugin_search_lag：索引最近写入时间 vs plugins.updated_at 最大值，超过 24h 判 degraded。
3. 搜索每次请求打一行结构化日志（src/server/platform/log.ts）：q_len / hits / duration_ms / fallback_used，不打 q 原文。
4. 业务数字写进 scripts/diagnose.mjs 的口径（health.ts:8 已经这样分工）。

---

## 9. 【入口】新增匿名高频只读端点，nginx 层没有任何闸门 —— 方案第 33-41 行

**① 哪部分有问题**：方案没有限流设计。

**② 后果**：
- docker/nginx/snippets/site-locations.conf:39-44 的 location / 直接 proxy_pass，没有 limit_req；docker/nginx/nginx.conf 全篇也没有 limit_req_zone。搜索是匿名、最容易被脚本化的高频只读接口。
- better-sqlite3 同步执行（db.ts:6-8）：并发不会被并行消费，只会排队；docker/nginx/snippets/proxy-common.conf:41 的 proxy_read_timeout 60s 意味着一个卡住的查询能占住连接 60 s。
- 现有 rate-limit.ts 的键是登录用户，匿名路径进不去 → 没有现成闸门可挂。

**③ 建议**：nginx 侧对该 location 加 limit_req（IP 维度；proxy-common.conf:14-23 已把 X-Real-IP 覆盖成权威值，键可靠），并显式 include proxy-common.conf（site-locations.conf:1-4 写明过的坑：新增 location 不 include 会让安全头悄悄消失）。

---

## 10. 【发布剧本】回填在发布序列里的位置没写 —— 方案全篇

**① 哪部分有问题**：没有「发布 / 回填 / 对账 / 回滚」四步。

**② 后果**：
- 不跑回填 = 存量插件全部搜不到，而新插件搜得到（scripts/backfill-review-search.mjs:5-6 原文称这是「最难解释的现象」）。方案没有回填，也就没有这条纪律。
- 重建是整表 DELETE，重建前必须先有恢复点，方案没写。
- 回填与每日 03:00 的备份（deploy/dsh-review-backup.timer）可能撞车，都是重 IO + 写锁。

**③ 建议**（照 0014 的分工，最小剧本）：
1. docker compose run --rm --no-deps web node scripts/backup.mjs --label pre-search（打恢复点）；
2. docker compose up -d → 确认 migrate Exited(0)、/api/health 含 fts.plugin_search；
3. 跑回填（分批），记录「写入 N / 耗时 / 索引体积」；
4. 打点验收：记忆、检索、检索测三类查询 + 一个不存在词 + 一个含双引号的词；
5. 回滚口径：代码回退 → 索引表保留（向前迁移，docs/docker约定.md:168）；前提是 §3 的自检项没被做成硬依赖，否则回滚动作本身会 502。

---

## 11. 与既有评审的关系（避免重复计分）

| 项 | 既有 m11-fulltext-search-SRE评审.md | 本文 |
|---|---|---|
| 中文分词 / unicode61 | §1 已写 | 不重复；并入「照 0014 复制」（§1） |
| 无重建入口 | §2 已写 | 补「分批提交 + 单实例锁 + 重建前恢复点 + 与备份撞车」（§4、§10） |
| 非原子 upsert | §3 已写 | 本文立场更保守：索引做成可修复派生数据，不要绑进发布事务（§4） |
| 注入 / 无 LIMIT / 限流 | §4、§10 已写 | 补「对备份恢复的威胁」与「query 不进访问日志」（§5） |
| 新表登记 REQUIRED_TABLES | §5 主张必须登记 | 本文反对无条件登记：回滚或止血会变成全站 502（§3） |
| 备份清单 / 磁盘 | §6 已写「白名单硬编码」 | 补 synchronous=FULL 与「磁盘-健康-写入三信号耦合」（§6） |
| 路径冲突 | §11 已写 | 补运行时 LIKE 兜底的具体落点与验收方式（§7） |

**一句话给作者**：这个仓库对「检索索引」已经有全套既成答案（0014 + search/index.ts + backfill 脚本 + health 扩展点）。方案的每一个 SRE 风险，都能在这里找到一份已经写好、已经踩过、已经留了注释的对照实现；照抄 0014 的工作量，远小于重新设计，且能直接继承它的运维经验。
