# N-60「用户数据自助导出」方案评审 · 问题清单

评审对象：tasks/n60-user-export/environment/proposal.md（共 24 行）　仓库：dsh-plugin-reviews-new　结论：**不通过**，需重写后再评。

一句话：方案的三个前提（reviews 有 author_id、comments 存正文、一次性 SELECT 可接受）全部不成立，且缺鉴权、频控、审计、异步化与范围表。

## 一、P0（照方案实现即事故）

### P0-1 §1 GET /api/me/export（proposal.md:10,13）
- **后果**：方案把「导出」定成 GET：一次响应里直接带全部用户数据。GET 是浏览器预取、地址栏、后退缓存、链接分享都会触发的动词，而这个响应装着用户的全部个人数据；链路还会经过 nginx 与浏览器缓存层，触发导出本身不可见、不可审计。项目约定是「结果由状态码表达 + 写操作走 requestKey」(docs/API端点与路径.md:16-21)，且 /media 之所以禁缓存就是因为「可见性由应用判定，缓存会绕过权限」(docker/nginx/snippets/site-locations.conf:29-36；src/app/media/[attachmentId]/[variantKey]/route.ts:44-48)。
- **建议**：改成 POST /api/me/export 创建导出作业（202 + taskId），下载走独立 GET /api/me/export/{taskId}/download 并带 `cache-control: private, no-store` + `content-disposition: attachment`；创建必须带 requestKey 做幂等（src/server/platform/request-id.ts:8-14 明确 request_id 不能当幂等键）。

### P0-2 §1 `SELECT * FROM reviews WHERE author_id = ?`（proposal.md:13）
- **后果**：两处错误叠加。(a) 前提不成立：reviews 表没有 author_id 列（migrations/0001_baseline.sql:149-158，只有 id/plugin_id/source_kind/published_revision_id/latest_submitted_revision_id），作者在 content_entities.author_id（同文件:107；索引 :146），且 reviews.id = content_entities.id（:104-145，含 UNIQUE(id,author_id)）——SQL 直接 `no such column: author_id`，接口 500。(b) 就算把 WHERE 迁到 content_entities，SELECT * 仍拿不到正文：title/body_markdown/submitted_at 在 review_revisions（:164-178），SELECT * 只返回 reviews 的 5 列，结果是一条『成功但空壳』的导出。
- **建议**：写死 join 路径：`reviews r JOIN content_entities ce ON ce.id = r.id LEFT JOIN review_revisions rev ON rev.id = r.published_revision_id`，并逐表列白名单列；补回归断言『导出的 reviews[] 里 body 非空』（防的正是空壳成功）。

### P0-3 §1 `SELECT * FROM comments WHERE author_id = ?`（proposal.md:14）
- **后果**：字段名碰巧对（migrations/0001_baseline.sql:255 author_id 存在），但 comments **没有正文**——正文在 comment_revisions.body（:284-292），comments 只持 published_revision_id/latest_submitted_revision_id 两个指针（:259）。这条 SQL 跑通仍导出评论骨架；而且它与 P0-2 中『reviews.author_id 不存在』相互矛盾，两条同样写法的 SQL 一个能跑一个报错，评审与实际实现很容易按其中一个推出错误结论。
- **建议**：补 join comment_revisions，并明确取 published_revision_id（公开版）还是 latest_submitted_revision_id（最后提交版）——两者语义不同，必须写理由，否则同一份导出在不同实现人手里内容不同。

### P0-4 §2『JSON 格式，包含所有数据』（proposal.md:19-20）
- **后果**：`SELECT *` + 原样 json 会把属于别人或平台管理的信息吐给用户：comments.hidden_by（migrations/0012_interactions.sql:89，管理员 userId）、forum_posts.deleted_by（src/server/modules/forum/posts.ts:115,479，删除者 userId）、users 整行（0001_baseline.sql:22-33，role/account_status/management_version/avatar_url 外链）。本仓已有同型事故判据：`SELECT *` 导出把 deleted_by（版主 userId）带出去被认定为『泄露他人信息』（docs/实测记录-M10-M13批量扩题.md:123-126）。
- **建议**：改成显式白名单列（形态可参考 docs/M13素材-用户数据导出v2.md:24-44 的 EXPORT_SPEC），并在文档里写三条负向断言：不出现 hidden_by/deleted_by/hidden_reason/管理字段；跨用户信息只以展示用昵称出现；内部 ID 不外溢。

### P0-5 §1 路由鉴权（proposal.md:10-16 全文无鉴权行）
- **后果**：`ctx.userId` 来源未定义，照字面实现会 500 或匿名可访问。项目有硬约定与回归用例：/api/me 路由第一步 `currentContext()` + `assertCanWrite(ctx)`（src/app/api/me/notifications/route.ts:21-24），顺序『先认人再读 body』已被测住（tests/auth-order.test.ts:1-13）。另外 assertCanWrite 是**写操作**前置校验（src/server/platform/context.ts:27-43），GET 用它语义错位，且会连带把禁用账户挡在门外——正是历史上 M13 被点出的『禁用账户被拒、合规上应允许』（docs/实测记录-M10M12M13-v2重出.md:114）。
- **建议**：用 currentContext() 取身份 + 独立的读侧守卫（只要求 userId 非空），禁用账户是否放行必须显式二选一并写进文档；补 auth-order 风格断言：匿名 → 401（不是 400/500）。

## 二、P1（能跑但会拖垮站点 / 违反仓内既有机制）

### P1-1 §1 一次性 `.all()` 全量查询（proposal.md:13-16）
- **后果**：better-sqlite3 是同步驱动，连接是进程级单例（src/server/platform/db.ts:66-74），web 进程只有一个 Node 主线程——`.all()` 物化整个结果集期间整站所有请求被挡住。本仓已把这个成本算过：『better-sqlite3 .all() 一次性物化整个结果集 → 数分钟内 Node 主线程 100% 占用，无法响应任何请求』（docs/实测记录-M10M12M13-v2重出.md:148-158）。再叠一层：`return json(...)`（proposal.md:16）会把整个对象再序列化一遍，两份内存同时在线。
- **建议**：导出异步化：POST 建 background_tasks（migrations/0001_baseline.sql:345-385），worker 分批（按 id 游标或 LIMIT/OFFSET）流式写文件，产物落 `/app/.data/exports`（该卷 web 与 worker 都挂了，docker/compose.yaml:64-67,91-92），配 TTL 与清理任务。

### P1-2 §1 无频控（proposal.md:10-16）
- **后果**：rate-limit.ts 的动作枚举里没有 export 动作（src/server/platform/rate-limit.ts:46-55 九项，无 me_export），方案也未提。导出是『全量拿走』的高价值操作，当前形态又是一次 DB 全表扫：任一登录用户反复调用即可把 web 进程打到不可用（同步驱动，见 P1-1）。
- **建议**：加 `me_export` 动作（建议 1 次/天 + 并发 1），并同步三处：RateLimitAction/RATE_LIMIT_RULES（rate-limit.ts:46-134）、settings 的配置键表（src/server/modules/settings/index.ts:51-138）、docs/API端点与路径.md:54-63 频控表——漏任一处就出现『后台能改但代码不认』。

### P1-3 §1 同步导出（proposal.md:10-16）
- **后果**：nginx `proxy_read_timeout 60s`（docker/nginx/snippets/proxy-common.conf:39-41）。P1-1 已算出导出是分钟级，用户拿到的是 504，而服务端仍会把查询做完——用户以为失败、库白干，重试叠加。site-locations.conf:41 那句『大响应（导出/长列表）不要撑着内存』说明前人对该场景有预期，但只调了 proxy_buffering，时长问题没解决。
- **建议**：异步化后此条自然消失（下载走文件、秒级）；若保留同步路径，必须写清超时与失败口径，并明确禁止调大 proxy_read_timeout 来『消炎』——那只会延长阻塞窗口。

## 三、P2（合规与范围，决定这东西到底算不算『导出』）

### P2-1 §1/§2 覆盖范围，以及第 24 行『满足数据可携带权』（proposal.md:13-24）
- **后果**：只导 users/reviews/comments，而属于个人的数据远不止：草稿 content_drafts(owner_id)（0001_baseline.sql:187-206）、收藏与收藏夹（0012_interactions.sql:22-40、0019_favorite_folders.sql）、整合包/帖子收藏（0039_favorite_folders_target_kind.sql:67-95）、论坛帖与回复（0037_forum.sql:71-153）、通知（0008_notifications.sql:19）、反馈工单（0036_feedback_tickets.sql:19）、纠错与申诉、批量导入上传（0031/0032）、偏好（0030_user_preferences.sql:19）。在此覆盖面上写『满足数据可携带权』属于结论超出证据。
- **建议**：把结论降级为『本人内容的一期版本』并给二期范围表；或此期就逐个列出 owner 列与 join 路径（注意各模块命名不统一：reviews 走 content_entities.author_id，forum_* 走 author_id，pack_reviews 另有作者列）。

### P2-2 §2 格式（proposal.md:19）
- **后果**：只写『JSON』不够：可携带权要求结构化、通用、可机器读取，且导出文件必须有版本与时间戳才能跨越后续表结构变更。本仓 JSON 已固定承担响应信封语义 `{ok,data,request_id}`（docs/API端点与路径.md:8-21），若导出文件复用它，用户拿到的是一份『接口响应』而不是『数据文件』，加字段/加版本无处安放；不复用则同一路径出现两种 JSON 形状。同题材 v2 素材已补了 schema_version + 导出时间戳（docs/M13素材-用户数据导出v2.md:59-60），本方案没有。
- **建议**：导出文件自带 `schema_version`/`exported_at`/`subject_id`/`counts` 头，并声明**不走** {ok,data} 信封；另补一节说明媒体附件（origial 只对本人可读，docs/API端点与路径.md:139）在导出里只给引用、不给字节。

### P2-3 §1 可见性口径缺失（proposal.md:13-16）
- **后果**：方案不过滤任何状态。被撤回（content_entities.withdrawn_at，0001_baseline.sql:117-118）或管理员隐藏（hidden_at/hidden_by/hidden_reason，0012_interactions.sql:88-90）的内容会连处置理由原文一起打包给作者；本人软删除的评论（deleted_at，0001_baseline.sql:263）也会被带走。同题材 v2 素材走了另一个极端——套 publiclyVisible 过滤（docs/M13素材-用户数据导出v2.md:46-51），对『导出自己的数据』恰恰是错的（未发布内容也有权拿走）。两头都错，方案没表态。
- **建议**：给一张分状态口径表并写理由：本人内容一律导出（含草稿/待审/撤回）；hidden_reason/hidden_by 不导出；本人主动删除的不再导出，管理员删除的导出内容但不导出删除者。

### P2-4 §1 评论回复关系（proposal.md:14）
- **后果**：comments 的 reply_to_comment_id/root_comment_id（0001_baseline.sql:256-279）指向**他人**评论。为『可读性』补上被回复内容片段就是把他人内容带走；不处理则回复树断裂。此外该联合外键是 DEFERRABLE(:276-279)，父子评论允许同事务乱序插入，导出侧不能假设父在子先。
- **建议**：只导出本人的 reply_to_comment_id 原值（跨用户引用一律降级为 ID），可读性必要时另给同属本人的父评论快照，并在文档里写明这条降级规则。

### P2-5 §1 无审计（proposal.md:10-24 全篇未提）
- **后果**：导出等于全量拿走个人数据，属高价值操作；operation_records 是本仓审计载体（0001_baseline.sql:411-433，action_type/operation_key/actor_kind/target_snapshot 齐备，UNIQUE(action_type,operation_key) :426），管理员改身份即在同一事务内写它（src/server/modules/identity/admin.ts:25,292）。缺了这一步，事后查不到『谁何时导走全部数据』，也没有可复用的去重键。
- **建议**：每次真实创建导出都写一条 operation_records（actor_kind='user'，target_type='export_job'，operation_key=request_key，reason=用户自助导出）；下载另记一条。这样 P0-1 的幂等与审计共用同一把键。

### P2-6 §1 `getUser(ctx.userId)`（proposal.md:12）
- **后果**：全仓没有 getUser，只有 private 的 findUser（src/server/modules/identity/service.ts:126-130，且只取 id/nickname/avatar_url/role/account_status 五列，不是整行）。方案用了不存在的 API 名，实现时极易被随手写成『什么都取』的新函数——正好打开 P0-4 的口子。
- **建议**：写明复用 createIdentityService 或新增**白名单** getter（仅 nickname/avatar_url/created_at），并注明『新增，不复用现成整行查询』。

## 四、跨条目的共同根因

1. **方案是照着『直觉上的一张表』写的，没对着 migrations/ 核对列名**：reviews 无 author_id（0001_baseline.sql:149-158）、comments 无正文（:284-292）、正文在 review_revisions——三条 SQL 里有两条跑不通、一条跑通但空壳。评审时若只核字段名不核『字段在哪张表』，会漏掉后半截。
2. **单条同步请求扛全量导出**：better-sqlite3 同步驱动 + 单线程 + 单连接（db.ts:66-74），这一条同时触发了事件循环阻塞（P1-1）与 nginx 60s 超时（P1-3）。异步导出不是优化，是这个技术栈下的正确性前提。
3. **『导出』被当成读接口写**：于是没有 requestKey 幂等、没有审计、没有频控、没有产物生命周期；而本仓对『一次性、可重放、高价值』的操作已有成套范式（op 记录 + 限流 + 后台任务），方案一条都没接。

## 五、建议的返工顺序（给作者）

1. 先改 §1 的 SQL：三张表按 migrations 重写为 join + 白名单列（P0-2/3/4/P2-6）；
2. 再定形态：POST 建作业 + 异步产物 + 独立下载地址 + requestKey 幂等 + 限流 + 审计（P0-1/P0-5/P1-1/P1-2/P2-5）；
3. 最后补两张表：导出范围表（每张表的 owner 列与 join 路径，P2-1）与可见性口径表（撤回/隐藏/本人删除怎么处理，P2-3）；
4. 格式补 schema_version 与头字段，并说明媒体附件只给引用（P2-2）；
5. 结论句从『满足数据可携带权』降级为可证伪的表述（P2-1），否则评审无法收口。
