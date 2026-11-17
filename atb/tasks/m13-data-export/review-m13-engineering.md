# 评审：用户数据导出方案（M13 / proposal.md 65 行）

L1 直跑：只读检索（Select-String/Get-Content 于 src、migrations、docker、tests）— 风险=零（未改任何文件）/ 频次=本次约 30 条合并留痕 / 可逆性=天然可逆。

按工作纪律第1条: 理论先行——先查相关理论与既有实践再动手。
按工作纪律第2条: 全面处理——先读任务/工作记忆, 有相关成果就在其上开发, 没有则先设计调研。
按工作纪律第3条: 白箱方法——用认知图【识别条件→找知识规则→精准执行】, 正确记录/错误找条件/不猜测/未验证不写入。
按工作纪律第4条: 根因纪律——先取证现场不猜测; 根因找到并验证后归档认知图并补不适用条件。
按工作纪律第5条: 验证纪律——入库前必须过回放/断言/回归验证, 未验证不固化。
按工作纪律第7条: 兜底纪律——主路径不可用则切等价兜底(如MCP→python直调), 并写进文档。
按工作纪律第8条: 中文思考——中文书写; 复杂(新奇/挑战/曾受批评/高信息差)→「我们需要」交流确认互补盲区; 简单→「让我」快速执行不空耗; 每段推理开篇用中文短语锚定语言。
按工作纪律第11条: 历史查询优先——先查记录/源码用已走通算法, 未找到不盲选DEFER；线稿用darkline(Sobel+mean+std*1.5)白底黑线, 拒raw canny/zone_linework。
按工作纪律第16条: 任务收尾归档——每次任务执行完只提炼核心修改(内容/原因/位置/验证结论)并按 CCG 六要素成文存入灵枢记忆。
按工作纪律第17条: 任务派发统一走蜂巢——… L1只读判定(产物落点=不改仓库/外部状态)可直跑, 须输出「L1 直跑：<命令> — 风险/频次/可逆性」留痕。
按工作纪律第18条: 工作区索引优先——查工作区文件先读 WORKSPACE_INDEX.md(仓根, 管线生成); 无则先跑 scripts/workspace_index.py --write 生成再读, 不以重复全盘浏览代替。

> 第18条现场：仓根无 WORKSPACE_INDEX.md，仓内也**没有** scripts/workspace_index.py（scripts/ 只有 mjs 运维脚本），无生成器可跑；本次为「已明确路径 + 单文件评审」，按该条不适用边界走 glob/read+grep 定向取证，未做全盘浏览。

## 一、结论

方案作为「草图」只覆盖了接口形状，**工程上不可直接实现**：身份检查入口选错、导出主体表选错、可见性判据与合规目标相互抵消、产物路径与部署前提不成立、后台任务这一层最关键的东西（任务类型/队列语义/失败语义）整段缺失。按现状实现会同时产生：①合规承诺不成立 ②导出目录无限增长 ③下载侧越权面 ④生产环境 HTTP 500。以下 22 条，按严重度排序。

---

## 二、P0（阻断：按方案写下去会出错或出安全事故）

### 1. §2 身份检查调用了错误的函数签名 → POST 接口在生产是 HTTP 500
- **问题**：`await enforceRateLimit({ action: 'me_export', ctx })`
- **后果**：真实签名是 `enforceRateLimit(db, action, userId, points?, override?)`（src/server/platform/rate-limit.ts:196-208）。按方案传一个对象，`db` 拿到的是对象字面量 → `limiterFor()` 里 `this.limiterCache.get(db)`（:157-159）对同一个对象反复 new `RateLimiterSQLite`，很快抛 TypeError；异常被 `errorResponse` 折成 `internal` 500（src/server/platform/errors-http.ts:141-145）。正确用法见 src/app/api/reports/route.ts:46-52。另外 `action: 'me_export'` 不在 `RateLimitAction` 联合（rate-limit.ts:46-55）里，`RATE_LIMIT_RULES` 是 `Record<RateLimitAction, RateLimitRule>`（:68），不加就 **typecheck 红**。
- **建议**：`const ticket = await enforceRateLimit(db, 'me_export', ctx.userId, 1, await rateLimitOverrideFor(db, RATE_LIMIT_RULES.me_export))`；把 `me_export` 同时加进 `RateLimitAction`、`RATE_LIMIT_RULES`、settings 的 `SETTING_DEFINITIONS`（src/server/modules/settings/index.ts:49+ 现有 9 项频控键逐字对应 `specKey`）；幂等重放/失败分支调 `refundQuietly(ticket)`。

### 2. §2 用 `assertCanWrite` 做「导出自己的数据」的门 → 最该导出的两类用户被挡死
- **问题**：`assertCanWrite(ctx)` 的语义是「写入前置」。
- **后果**：(a) 匿名 → 401 合理，但**导出是读操作**，平台层没有任何「本人读取自己数据」的判据，方案却拿写入判据顶替；(b) **账户被禁用（account_status='disabled'）→ `err.disabled()` 403 'account_disabled'**（src/server/platform/context.ts:34-43 + errors-http.ts:34）——而账户被禁用/即将被处置的时刻恰恰是用户最需要取走自己数据的时刻；(c) 禁用在同一事务里还会撤销该用户全部会话（src/server/modules/identity/admin.ts:567-571），`resolveSession` 对已撤销会话直接 return null（src/server/modules/identity/service.ts:245-276）→ `currentContext()` 退化成 anonymous（src/app/api/_lib/context.ts:21-28）→ 连「先登录取数」都不可能；(d) 全仓没有任何账户注销/删除通路（grep `DELETE FROM users`/「注销」零命中），方案也没定义「被禁用者如何行使可携带权」这条线。
- **建议**：新增 `assertCanReadOwnData(ctx)`（只要求 userId 非空，忽略 account_status），并对「已禁用/即将注销」的账户给出**带外通道**（管理员协助导出或邮件一次性取数链接）；接口文件头显式说明这是与 `assertCanWrite` 不同的判据，防止后人「统一」回写入判据。

### 3. §3 reviews 的导出主体选错（从末层表选，而不是从载体表选）
- **问题**：`table:'reviews', columns:['id','plugin_id','source_kind','created_at'], ownerColumn:'content_entities.author_id', join:'JOIN content_entities ce ON ce.id = reviews.id'`
- **后果**：(a) 选的列里 **没有正文**——评测的全部可见内容在 `review_revisions`（migrations/0001_baseline.sql:164-178：title/body_markdown/plugin_version/dsh_version/assessment_level），导出的「评测」只有 id+插件 id+创建时间，**用户拿到的不是自己的文章**；(b) `reviews` 有两列可空外键 `plugin_id`（:152）与 `CHECK (published_revision_id IS NULL OR plugin_id IS NOT NULL)`（:156-157），草稿期创建的 review 行 `plugin_id` 可能为 NULL → 用 INNER JOIN 会**静默丢行**，而导出最忌讳静默少数据；(c) 草稿（content_drafts，:187-206）完全不在范围内，而用户「写了一半的东西」正是可携带权最该覆盖的。
- **建议**：主体改成 `content_entities ce` 左连 `reviews`（`LEFT JOIN`），再按 review 展开 `review_revisions`（全部 revision_number，含未发布的最新提交修订），并单独列 `content_drafts`；导出值用显式 NULL 语义，禁止依赖 INNER JOIN 的隐性过滤。

### 4. §4 可见性过滤与「数据可携带权」互相抵消（最严重的语义错误）
- **问题**：`if (!resolveVisibilityInTx(db, row.id).publiclyVisible) continue;`
- **后果**：`publiclyVisible = 有公开修订 && 未撤回 && 未隐藏`（src/server/modules/content/index.ts:153-166、367-369）。导出时把「作者**自己撤回**」「管理员隐藏」「从未发布」的条目全部丢掉 = 用户拿不到自己写过的内容，**恰恰违反可携带权**。更狠的是撤回链路本身：`deleteDraft` 对已发布评测是「先撤回再删草稿」（src/server/modules/reviews/index.ts:771-787），也就是说**用户每删一篇，那篇就永久从自己的导出里消失**——「删除」被实现成了「不可恢复且不可导出」。
- **建议**：导出**不按公开性过滤**；改为「全量导出 + 每条带 `visibility` 字段（published / withdrawn(含 withdrawal_reason) / hidden / draft）」。若确实要遮蔽第三方信息（例如管理员隐藏原因里的内部说明），只遮蔽**他人**的信息，不遮蔽用户本人的正文。

### 5. §3 forum_posts 白名单列缺 `author_id`，却拿它当归属列；且软删除帖被过滤
- **问题**：`ownerColumn:'author_id'`，但 `columns` 里没有 `author_id`。
- **后果**：`author_id` 确实存在（migrations/0037_forum.sql:74），能在 WHERE 用、不必进输出；但方案把「白名单列」与「归属列」混在一张 spec 里，实现者极易把 ownerColumn 当输出列 → 导出里带出内部列，或者反过来在 SELECT 白名单里漏用它导致 SQL 直接报 no such column。同时 `status='deleted'` 的帖子（0037:77、:87-89）在「按页面判据过滤」下被排除——用户看不到「我发过又删掉的帖子」，与第 4 条同源。
- **建议**：spec 显式区分 `selectColumns` / `ownerColumn` / `stateColumn`，ownerColumn 强制参与 WHERE 但**不自动参与输出**；deleted 帖子导出为 `{status:'deleted', deleted_at, deleted_by_self}`，不打码。

### 6. §1/§3/§5 完全没提 `background_tasks.task_type` —— 异步任务在这一层根本入不了库
- **问题**：`POST /api/me/export → 202 { taskId }`，但没说这个任务是什么类型。
- **后果**：`task_type` 是**不可配置的注册集合**：`field-rules.ts:87-96` 只有 5 个（review_check / review_publish / comment_check / comment_publish / plugin_discovery），SQLite 里是硬 CHECK（migrations/0007_task_type_plugin_discovery.sql:25-26）。不加类型 → `INSERT` 触发 CHECK 失败；加了类型却不加 handler → `createWorker.execute` 判 `no_handler`、任务以 **skipped** 终结（src/worker/main.ts:88-104），接口返回的 202 永远查不到结果。而且「加类型」不是改常量就完：得新写一条 0043 重建表（0007 的形状：`dsh:migration-transaction=off` + `PRAGMA foreign_keys=OFF` + 手工 BEGIN/COMMIT），它会同时**卡红两条测试**：`tests/migrate-transaction.test.ts:99-111`（库 CHECK 必须与 `TaskTypes` 逐条一致）与 `tests/worker-handlers.test.ts:922-933`（每个 TaskType 必须注册 handler，且 handler 数量相等）。
- **建议**：方案补一节「队列接入」：新增 `user_data_export` 到 `TaskTypes` + `RATE_LIMIT_RULES`/settings + 0043 迁移（照 0007 的写法）+ `handlers.ts` 注册 + 两条现有测试纳入验收清单。否则 202 是假承诺。

### 7. §5 产物路径 `/data/exports/...` 在部署里不存在，且会破坏「除 /root 外不留痕」与备份口径
- **问题**：`/data/exports/<userId>/<taskId>.json.gz`
- **后果**：(a) 容器里没有 `/data`：web 只挂 `${DSH_REVIEW_DATA_DIR}:/app/.data`（docker/compose.yaml:64-67）、worker 同样（:91-92），`/data` 是**容器可写层**——容器重建（`docker compose up -d --build`）产物全丢；(b) 若改用镜像内路径写入，违反 docs/docker约定.md §1「数据（库+媒体）只在宿主 /root 下的一个目录」；(c) 备份只覆盖 `.data` 与 `media`（docs/docker约定.md:160-165），导出产物不进备份也不该进——但方案没说明「产物不是数据」，运维会误以为丢了要恢复；(d) 落盘属主必须与容器 UID 10001 一致（compose.yaml:20-21、docker约定.md:94-101），方案也没提。
- **建议**：产物根走独立 env（如 `EXPORT_ROOT`，默认 `${DSH_REVIEW_DATA_DIR}/../exports`，compose 里单独挂一个卷），并在 docs/docker约定.md §3 登记为「运行期临时数据，不进备份、可随时删除」；目录权限与 UID 一并写进方案。

### 8. §3 写死的 `/api/me/export/:taskId/download` 是越权面最大的一条路径，方案没有任何隔离要求
- **问题**：下载接口只给「文件」两个字。
- **后果**：`<userId>/<taskId>.json.gz` 是**静态路径**，一旦有人把产物目录挂进 nginx/静态服务（仓库对媒体的规矩是「只反代、不直读磁盘」docs/docker约定.md:15），链接即全量泄露。仓内已有正例可参照：`/media/[attachmentId]/[variantKey]` 有专门鉴权路由（src/app/media/[attachmentId]/[variantKey]/route.ts:30-49）。方案里下载路径既没要求「校验 taskId 属于当前 userId」也没要求「每次下载重新认人」。
- **建议**：下载必须走 handler 内身份检查：①按 `(taskId, ctx.userId)` 查任务，属主不符一律 404（不区分「不存在」与「无权」，与 media 路由同口径）②下载时重新 `currentContext()`，禁 token-in-URL（仓内会话只走 Cookie：src/server/platform/session-cookie.ts）③产物文件名用服务端生成的随机名，永不复用可达 URL；④响应头 `content-disposition: attachment` + `cache-control: private, no-store`。

---

## 三、P1（严重：能跑但会出问题）

### 9. §5「TTL 7 天 + 定时清理」在仓库里没有可用的定时器
- **问题**：只说「定时清理」。
- **后果**：仓库明确拒绝长驻定时器：rate-limit.ts:182-184「不开数据库的定时清理：它会 setInterval，而我们要进程能被干净关掉」；全站唯一的周期逻辑挂在 worker 主循环上（src/worker/main.ts:170-206，每 `IDLE_SLEEP_MS`=10s 走一次 `runOnce`），web 进程没有任何定时任务能力。方案不指定清理承载者 → 要么没人清理（磁盘涨死），要么有人给 web 加 setInterval（违反既有约定、破坏优雅退出）。
- **建议**：清理挂在 worker 的 tick 上（与 `ensureDailyDiscoveryTask` 同款，src/worker/scheduled.ts），并给出「产物最长存活 = TTL + 一个 tick 间隔」的标准；清理失败只记日志、不中断主循环。

### 10. 没有导出任务的状态机 / 失败语义 / 重试成本分析
- **问题**：`GET /api/me/export/:taskId → 状态`，未定义状态枚举与失败呈现。
- **后果**：队列的既有语义是——**可重试**（error）会按 [30s,120s] 重试到 3 次（src/server/modules/tasks/constants.ts:34-64），**不可重试**（skipped）直接终结（handlers.ts:91-94）。一个「全量导出」跑 3 次等于把同一个用户的全表扫描做 3 遍，且在 `MAX_CONCURRENT_TASKS=4` 的全站并发池里（constants.ts:16）占坑——导出是长任务，会让评测/评论发布排队（worker 串行执行一批 claim，main.ts:199-205）。方案对「失败后用户看到什么」「重试要不要也发通知」零描述。
- **建议**：明确 outcome 映射（磁盘满/序列化失败=error 可重试；用户不存在/数据格式坏=skipped 终结），并在 `progress` 里写进度（`tasks.finish` 的 `progress` 是专为此留的结构化结果，service.ts:305-313）；给导出单独一条 `scheduling_group` 或在 claim 侧限制「全站同时最多 1 个导出」，避免挤占 4 个并发位。

### 11. 没有限流标准（额度、并发上限、幂等重放退还）
- **问题**：只有一句 `enforceRateLimit`。
- **后果**：频控模型是「每用户每动作每窗口 N 次」（rate-limit.ts:196-251），而导出的成本是「全表扫描 + 压缩 + 存到本地」，跟一次 INSERT 不是一个量级；`RateLimitTicket.refund()`（:142-149）在「同一请求重放」时**必须主动退还**，否则用户连点两下就把当天额度耗光。方案没给 points/duration，也没说重复提交同一导出要不要复用 taskId。
- **建议**：给 `me_export` 定「每用户每日 1-3 次 + 全站同时 ≤1 个在跑」两把闸；`deduplication_key` 用 `export:<userId>:<yyyy-mm-dd>` 让同日重复提交自然复用同一个 taskId（`tasks.enqueue` 的 UNIQUE(task_type, deduplication_key) 直接给到，src/server/modules/tasks/service.ts:184-226），并在重放分支 `refundQuietly(ticket)`。

### 12. 没有分页/流式与大小上限（内存与磁盘双无界）
- **问题**：§1「异步生成」之后没有取数方式。
- **后果**：单用户 5000 帖（body_md 上限 20KB，0037:76）+2000 评测修订（body_markdown 无长度上限，0001:170）按 `.all()` 全读进内存，worker 进程直接 OOM——而 worker 进程同时承载发布链路，OOM 等于全站内容发布停摆。另一方面产物无上限，一个恶意用户可用「写满再导」打爆磁盘，且小磁盘 VPS 上 `/app/.data` 与库同一个卷（compose.yaml:64-65），**写满磁盘会先打挂 SQLite 写入**（WAL 也在这个卷上）。
- **建议**：按 id 游标分批（`WHERE id > ? ORDER BY id LIMIT 500`）流式写 `createGzip()`，单条即写即弃；设「单次导出行数/字节上限」，超限就截断并在产物里写 `truncated:true + 总数`；写入前检查可用空间（预留 ≥2 倍预估大小）。

### 13. 没有审计留痕
- **问题**：§1/§5 全篇零审计。
- **后果**：导出是「批量搬走个人数据」的高价值动作。仓内已有现成的留痕设施 `recordOperationInTx`（src/server/modules/operations/index.ts，plugin_discovery 就是这样记的：src/worker/handlers.ts:318-351），方案不用它，等于事后无法回答「谁在什么时候导了多大一份」。注意 `operation_records` 是**业务审计**，不是导出专用表；且 0042 有不可删触发器（migrations/0042_audit_immutable.sql）。
- **建议**：导出开始时记一条 `user.export.request`（targetSnapshot=行数/预估字节）、结束时 `user.export.finish`（产物字节、耗时、行数、truncated），下载也要记一条（防止账号被盗后把数据一次性搬走而无迹可查）。

### 14. 产物链路同时踩了「单文件大小」与「nginx 下载路径」两个坑
- **问题**：§1 `GET .../download → 文件`。
- **后果**：仓库唯一的文件下发先例是媒体路由（整体 `new Response(new Uint8Array(bytes))`，src/app/media/[attachmentId]/[variantKey]/route.ts:39-49），它成立的前提是媒体单文件 ≤10MB（src/server/modules/media/service.ts:38）；导出包没有大小上限，`readFile` 整个读入再返回会二次放大内存。另外 `taskId` 如果用一个自增/短 id 就能枚举（仓内规范是 UUIDv7，field-rules.ts:14-18），枚举面进一步放大。
- **建议**：下载用 `createReadStream` + `Readable.toWeb()` 流式返回，内容长度用 `fs.stat`；taskId 用 UUIDv7；nginx 侧对该路径关闭 gzip（.gz 已压缩）并放宽 `proxy_read_timeout`，两个配置都要写进 docker/nginx。

### 15. 方案自称「符合数据可携带权」，缺口远不止 reviews/posts/favorites 三张表
- **问题**：§3 只 3 张表，§预期 却写「用户可下载自己的**全部**数据」。
- **后果**：仓库里用户本人的数据至少有：`comments`/`comment_revisions`（0001:252-292，评论也是用户创作）、`forum_post_replies`（0037:100-115）、`content_drafts`（0001:187）、`notifications`（0008 起多次重建）、`user_preferences`（0030）、`favorite_folders`/`pack_favorites`/`post_favorites`（0019、0039:67/80）、`review_helpful_votes`（0012:22）、`reports`（0012:42，用户提交的举报）、`feedback_tickets`（0036:19）、`role_applications`（0040:48）、`plugin_ownerships`（0040:82）、`attachments`/媒体文件（0003）、`plugin_correction_requests`（0015）。只导 3 张表就对外宣称合规，是把合规承诺写成一句空话。
- **建议**：把导出范围写成「显式清单 + 每项标注是否含本人正文/是否含第三方数据」，另加一节「**明确不导出**及其理由」（他人 id、内部管理字段、隐藏审核理由、会话 token），后者正是合规评审要看的。`user_sessions`（token 摘要）绝不能导。

### 16. 没有通知闭环（用户不知道导出好了/失败了）
- **问题**：`202 { taskId }` 之后只有轮询 `GET 状态`。
- **后果**：异步任务最典型的体验缺陷——用户关掉页面就再也拿不到结果；而 7 天 TTL 会让一个「第二天才想起来」的用户永久失去这次导出。仓内已有 notifications 模块（src/server/modules/notifications/index.ts），方案只字未提。
- **建议**：任务完成/失败各发一条站内通知（带 download 链接与过期时间），失败要写明原因与「是否可重试」；通知去重键参考 feedback 的 `feedback:{ticketId}:{recordId}` 写法（0036:56）。

### 17. §4 的调用形态与 `resolveVisibilityInTx` 的实际契约不符
- **问题**：`resolveVisibilityInTx(db, row.id)` 被当作「有返回值、可 continue」的行级判据。
- **后果**：该函数对**不存在的实体直接抛错**（src/server/modules/content/index.ts:91-100 `throw err.unavailable`），且每次调用要跑 3-4 条 SELECT（hasPublishedRevision 依次查 reviews / plugin_pack_releases / pack_reviews，:106-151）。若在导出循环里逐行调用：①任何一条 id 不在 content_entities 就整批 500；②N+1 查询把导出耗时放大一个数量级；③每次调用各开一次读快照，**同一份导出里不同条目来自不同时间点**（无快照一致性）。
- **建议**：先放弃「用可见性过滤」（见第 4 条）；确实要标注状态时，用一条 `LEFT JOIN content_entities` 的批量查询把 `withdrawn_at/hidden_at/published_revision_id` 一次取回、在内存里算状态，**不要**逐行调用解析器。快照一致性另行定义（见第 18 条）。

### 18. 没有「快照时点」定义
- **问题**：异步生成 + 7 天 TTL，但没写「这份导出里的数据是哪一刻的」。
- **后果**：worker 队列有 `available_at` 延后与 [30s,120s] 重试（constants.ts:34-39），真正执行可能比请求晚几分钟；导出过程中用户可能又写了新内容、撤回了评测。产物里只带「导出时间戳」（§6）而各表读取时点不一，等于一份没有一致性的混合快照，事后无法自证。
- **建议**：在请求时刻记录 `requested_at` 与队列执行时刻，产物里同时写 `snapshot_started_at/snapshot_finished_at` 与「闭区间内可能仍在变化」的说明；若可接受，用一条只读事务（`withRead`）包住整段取数以获得单一快照（代价：长事务推迟 WAL checkpoint）。

### 19. §6 格式约束不足以做到「可携带」
- **问题**：「JSON，带 schema_version 与导出时间戳」。
- **后果**：可携带权的实质是「**可移植**」，不是「可下载」。只有 schema_version 而没人保证 schema 稳定/有变更日志，下次改结构就等于把去年的导出变成孤儿文件；JSON 里的绝对毫秒时间戳（全站规范：field-rules.ts:6）对人类不可读，也没写时区；body_markdown 不带渲染规则会让读者拿到的是一坨源码。
- **建议**：产物里加 manifest（导出版本、涉及表清单、每表行数、字段说明与类型、时间单位与时区）+ schema 版本承诺；正文同时给 markdown 与已渲染 html（仓内已有 markdown 管线 src/server/platform/markdown.ts），或附一个可离线打开的 index。

### 20. 公测形态被忽略：讨论区总开关关闭时导出行为未定义
- **问题**：所有「我的 ××」接口都带开关判断；导出却同时包含 forum_posts。
- **后果**：`forum.enabled` 关闭时，讨论区接口一律 404、页面 notFound()（src/app/api/_lib/context.ts:30-45 的既定标准：「未开放的功能对外表现为不存在」）。导出现在会**通过导出接口把已关闭功能的帖子吐出去**，等于绕过开关；同时 `/api/me/forum-posts` 是带 `assertForumEnabled()` 的（src/app/api/me/forum-posts/route.ts:15），两条接口对同一份数据给出不同答案。
- **建议**：显式定义——开关关闭时导出**仍包含**用户自己的历史帖（更符合可携带权），但要在产物里保留「该功能当前未开放」的注记，并在接口文件头说明这是对开关标准的**有意例外**，避免下一个人「顺手统一」。

### 21. 依赖没有纳入验证：压缩/流式能力现成，但接口约定未定义
- **问题**：§5 用了 `.json.gz`，§1 用了 202。
- **后果**：(a) 全仓 `src/` 里**没有任何 zlib/gzip 使用**（grep 只命中 scripts/verify-task-prereqs.mjs:244 的白名单字符串），所以「gzip 产物」是全新能力，方案没说我方已有 node:zlib 可用（无需新依赖）、也没说压缩级别与 CPU 成本；(b) 202 与仓内惯例不齐：`jsonOk(data, requestId, status)` 支持自定义状态码（errors-http.ts:148-152），但现有写接口用 201（src/app/api/reports/route.ts:64），异步接口在仓内尚无先例，前端/验收脚本没有可参考的约定。
- **建议**：写明压缩用 `node:zlib` 的 `createGzip({level:6})` 流式写、预期压缩比；接口层补一条约定（202 + Location 指向状态接口），并同步到 docs/API端点与路径.md。

### 22. 方案与仓库/文档的既有约束冲突，且自身措辞不合本仓规范
- **问题**：全文用「鉴权」「落盘」「口径」等词；无迁移、无测试、无验收脚本。
- **后果**：docs/用词规范.md:30/33/34 明确要求 鉴权→身份检查、口径→标准、落盘→存到本地，并且有可执行门禁 `npm run check:wording`（scripts/check-wording.mjs，对 docs/ 下 .md 生效）；方案一旦落到 docs/ 就过不了门禁。另外方案涉及新的库结构（task_type 扩展）却没有迁移编号，也完全没提测试与验收脚本，而仓内正有两条硬测试守卫这个面（见第 6 条）。
- **建议**：按用词规范重写术语；补一节「迁移 + 测试 + 验收」：迁移 0043（照 0007 形状）、`tests/worker-handlers.test.ts` 注册表用例纳入、`tests/migrate-transaction.test.ts` 一致性用例纳入、新增 `scripts/acceptance-user-export.mjs`（与 acceptance-rate-limit.mjs 同款验收脚本）。

---

## 四、方案里「对的」部分（避免推翻重来）
- 异步生成而不同步返回，方向正确（大对象同步返回会卡住 Node 单线程）。
- 用白名单列而不是 `SELECT *`，方向正确（v1 素材 docs/M13素材-用户数据导出.md 就是 `SELECT *`，这一版是进步）；只是白名单内容不含正文，见第 3 条。
- 产物独立于库、TTL 清理，思路对；缺的是落点（第 7 条）与承载者（第 9 条）。
