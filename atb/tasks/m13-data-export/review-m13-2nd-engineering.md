# 评审：用户数据导出方案（M13 · proposal.md 65 行）

> 评审对象：`atb/tasks/m13-data-export/environment/proposal.md`（1544 字节，sha256 `ba46247adb3fcc5d…`，65 行，与 `docs/M13素材-用户数据导出v2.md` **字节级完全相同**）
> 仓库根：`dsh-plugin-reviews-new`（migrations 至 0042，Next 16 + better-sqlite3 同步驱动 + SQLite WAL + 单 worker）
> 评审立场：**工程可实施性 + 安全**。所有结论均落到 `文件:行号`，未运行代码。

L1 直跑：只读检索（read/grep 于 src、migrations、docker，约 30 条命令合并留痕）— 风险=零（未改任何文件）/ 频次=单任务 / 可逆性=天然可逆。

---

## 结论先说

这份方案是**用「读页面」的心智模型去写「搬数据」的功能**，它有三层自相矛盾：

1. **合规层**：§背景 承诺「符合数据可携带权 / 全部数据」，§3 只导 3 张表、§4 还把不可见的删掉——承诺与实现在语义上互相抵消；
2. **架构层**：§1 承诺异步（202 + taskId），但仓库的任务类型是**不可配置的封闭集合**，方案没有接入点，202 是空头支票；
3. **代码层**：§2 与 §3 的代码**照抄就编译不过 / 运行必报错**（频控签名错、`reviews.created_at` 不存在、`forum_posts` 直接抛错）。

按现状实现，交付的会是一个「请求成功返回 taskId → 任务永远 skipped → 用户永远等不到文件」的功能，同时把「假合规」写进对外承诺。以下 18 条，P0 8 条 / P1 7 条 / P2 3 条。

---

## 一、P0（按方案写下去必错，或构成安全事故）

### 1. §2 频控调用形态错误 → 接口一调用就 500

- **哪部分**：`proposal.md:21` `await enforceRateLimit({ action: 'me_export', ctx })`
- **后果**：
  - 真实签名是 `enforceRateLimit(db, action, userId, points?, override?)`（`src/server/platform/rate-limit.ts:196-208`）。按方案传**一个对象**：`db` 位拿到对象字面量、`userId` 位是 `undefined` → 命中 `:209-213` 的 `typeof userId !== 'string'` 检查 → 抛 `err.internal` → **HTTP 500**（不是 401/429）。
  - `'me_export'` 不在 `RateLimitAction` 联合里（`rate-limit.ts:46-55`，现有 9 个动作无此项），而 `RATE_LIMIT_RULES` 是 `Record<RateLimitAction, RateLimitRule>`（`:68`）→ **typecheck 直接红**，连编译都过不去。
- **建议**：`const ticket = await enforceRateLimit(db, 'export_request', ctx.userId)`；同时把 `export_request` 补进 `RateLimitAction`、`RATE_LIMIT_RULES`、以及 settings 的 `SETTING_DEFINITIONS`（`specKey` 与动作名逐字对应，见 `:59-60` 注释的约定）。参考范式 `src/app/api/reports/route.ts:46-52`。

### 2. §2 用 `assertCanWrite` 当「导出自己数据」的门 → 最该导出的用户被挡死

- **哪部分**：`proposal.md:20` `assertCanWrite(ctx);   // 先认人，挡匿名与禁用账户`
- **后果**：
  - 该函数的语义是**写入前置**，且明确要求 `accountStatus === 'active'`，否则抛 `err.disabled()`（`src/server/platform/context.ts:34-43`）。
  - **导出是纯读操作**，却挂在写入判据上：账户被禁用（`account_status='disabled'`）→ 403 `account_disabled`（`:41` + `isAuthError` 的 `:57`）。而**账户被禁用恰是用户最需要取走自己数据的时刻**，这与「数据可携带权」正面冲突。
  - 注释里「挡匿名与禁用账户」把这一取巧写成了设计意图，后续维护者不会怀疑——**错误的语义被文档化**才是最危险的。
- **建议**：新增 `assertCanReadOwnData(ctx)`：只要求 `ctx.userId !== null`，**不校验 account_status**；并在文件头写明「与 `assertCanWrite` 是不同判据，勿统一」。若产品坚持禁用即封禁，必须提供带外取数通道（管理员协助导出 / 邮件一次性链接）。

### 3. §3 `reviews` 的 `created_at` 列不存在 → SQL 直接 `no such column`

- **哪部分**：`proposal.md:28-31` `table: 'reviews', columns: ['id','plugin_id','source_kind','created_at']`
- **后果**：
  - `reviews` 表**只有 5 列**（`migrations/0001_baseline.sql:149-158`：id / plugin_id / source_kind / published_revision_id / latest_submitted_revision_id），**没有 `created_at`**。创建时间在父表 `content_entities.created_at`（`:114`），两者是 1:1 主键共享（`reviews.id REFERENCES content_entities(id)`，`:150-151`）。
  - 按 spec 生成的 `SELECT id, plugin_id, source_kind, created_at FROM reviews` → 运行时 **`no such column: reviews.created_at`**，整个导出任务失败。
  - 更隐蔽：补上 §3 的 `join` 后，`created_at` 会**歧义解析**到 `ce.created_at` 侥幸跑通；将来 `reviews` 若补此列，同一句 SQL **静默改指向另一列**，导出含义变了且不报错。
  - 仓库正例：必须 `JOIN content_entities ce ON ce.id = r.id`（`src/app/me/page.tsx:107-109`、`src/app/api/reviews/[reviewId]/route.ts:57`）。
- **建议**：spec 改为全限定列名 `['reviews.id','reviews.plugin_id','reviews.source_kind','content_entities.created_at']`；生成 SQL 前**校验每个列前缀必须是 spec 内已声明的表**，让这类错误在生成期失败而不是运行期。

### 4. §3 `reviews` 白名单列里没有正文 → 用户「导出自己的评测」拿到的只有 id

- **哪部分**：`proposal.md:28-31` 的 `columns`
- **后果**：评测的**全部可见内容**在 `review_revisions`（`migrations/0001_baseline.sql:164-178`：title / body_markdown / plugin_version / dsh_version），`reviews` 只是指针表。导出的「评测」= `id + plugin_id + source_kind + 时间`，**用户拿不到自己写的文章**——这已经不是「不合规」，是「导出功能没实现」。
- **建议**：主体从 `content_entities` 出发（`LEFT JOIN reviews` → 展开 `review_revisions` 全部 `revision_number`，含未发布的最新提交修订），另单列 `content_drafts`（`0001:187-206`，`owner_id`）。

### 5. §3 `reviews` 用 INNER JOIN + 可空 `plugin_id` → 静默丢行

- **哪部分**：`proposal.md:31` `join: 'JOIN content_entities ce ON ce.id = reviews.id'`（INNER）
- **后果**：`plugin_id` 可空（`0001_baseline.sql:152`），且 `CHECK (published_revision_id IS NULL OR plugin_id IS NOT NULL)`（`:156-157`）意味着**草稿期 review 的 plugin_id 可以为 NULL**。若 `reviews` 行尚未落库（只有 `content_entities` 行），INNER JOIN 直接把它丢掉。**导出最忌讳静默少数据**：用户不会知道自己少了什么，事后也无法自证。
- **建议**：以 `content_entities` 为左表 `LEFT JOIN`；产物里显式 NULL 语义，禁止用 JOIN 类型做隐性过滤；导出结束时把「每表应导行数 vs 实导行数」写进产物。

### 6. §4 可见性过滤与「数据可携带权」直接抵消（最严重的语义错误）

- **哪部分**：`proposal.md:50` `if (!resolveVisibilityInTx(db, row.id).publiclyVisible) continue;`
- **后果**：
  - `publiclyVisible = 有公开修订 && 未撤回 && 未隐藏`（`src/server/modules/content/index.ts:153-166`、判定定义 `:158-165`）。
  - 于是「作者自己撤回的」「管理员隐藏的」「从未发布的」**全部被过滤**。数据可携带权的对象是「用户提交过的全部个人数据」，**不是「当前公开可见的内容」**——按现方案，用户越是想取走被下架的内容，越是取不到。**方案自称合规，实际做法恰好违反它**。
  - 该函数对**不存在于 `content_entities` 的 id 会直接 throw**（`:91-100` `readControlRow` → `err.unavailable`）——见下条。
- **建议**：**导出不做公开性过滤**。改为全量导出 + 每行带 `state: 'published' | 'withdrawn' | 'hidden' | 'draft'` 与 `withdrawn_at / withdrawal_reason / hidden_at`。若必须遮蔽他人信息，只遮蔽**审核内部说明**，绝不遮蔽用户本人的正文。

### 7. §4 的可见性判据套不到 `forum_posts` 上 → 导出任务必崩

- **哪部分**：`proposal.md:50` 对**所有表**统一调用 `resolveVisibilityInTx(db, row.id)`
- **后果**：`forum_posts` 不是 `content_entities` 的子类型（`0037_forum.sql:71-90`，独立主键表）。把帖子 id 传给该函数 → `content_entities` 查不到 → `err.unavailable('内容不存在或已不可访问。')`（`content/index.ts:91-100`）→ **整个导出批次 500**。也就是说：只要用户发过一条帖子，导出就永远失败。
- **建议**：可见性按表分别定义（评测看 `content_entities` 三列，帖子看 `forum_posts.status`），并在 spec 里为每张表声明自己的 `stateColumn`；**不要**在循环里调度一般化的解析器。

### 8. §5 产物路径 `/data/exports/...` 在部署里不存在

- **哪部分**：`proposal.md:55` `/data/exports/<userId>/<taskId>.json.gz`
- **后果**：
  - 全仓 `docker/` 里**没有任何 `/data` 挂载**；容器只挂 `${DSH_REVIEW_DATA_DIR}:/app/.data`（`docker/compose.yaml:41, 65, 92`）。写到 `/data` = 写进**容器可写层**：`docker compose up -d --build` 重建即全部丢失，而这里恰好是「7 天内可取」的承诺。
  - `docs/docker约定.md` 的既定口径是「数据只在宿主一个目录下」，方案没说明这份产物属不属于「数据」。
  - 容器以固定 `10001:10001` 运行（`docker/web.Dockerfile:50-55, 71`；`compose.yaml:35, 50, 85`），新建 `/data` 若属主不符 → **`EACCES`**。
- **建议**：产物根走独立 env（如 `EXPORT_ROOT`，默认 `${DSH_REVIEW_DATA_DIR}/../exports`），compose 里**单独挂一个卷**；在 `docs/docker约定.md` 登记为「运行期临时数据：不进备份、可随时删除」；目录属主与 UID 写进方案。

---

## 二、P1（能跑但会出问题）

### 9. §1/§3 没有 `task_type` 接入点 → 202 是空头承诺（本节最关键）

- **哪部分**：`proposal.md:10` `POST /api/me/export → 202 { taskId }`，以及全篇未提任务类型
- **后果**：
  - `TaskTypes` 是**不可配置的封闭注册集合**，注释明确写着「不能由配置注入新类型」（`src/packages/common/field-rules.ts:87-95`，现仅 5 个）。
  - 数据库层是**硬 CHECK**：`task_type IN (...)`（`migrations/0007_task_type_plugin_discovery.sql:26`，最新一条；`0001_baseline.sql:348` 是旧版）→ 不加类型，`INSERT` 直接触发 CHECK 失败。
  - 加了类型但不注册 handler → `createWorker.execute` 判 `no_handler`，任务以 **skipped** 终结（`src/worker/main.ts:88-104`）→ 用户拿到的 202 永远查不到结果。
  - 加类型也不是改常量：需写 **0043 迁移重建 `background_tasks`**（照 0007 的写法：`dsh:migration-transaction=off` + `PRAGMA foreign_keys=OFF` + 手工 BEGIN/COMMIT，见 `src/server/platform/migrate.ts:47` 的说明）。
- **建议**：方案补一节「队列接入」：①`TaskTypes` 增 `user_data_export` ②0043 迁移重建 CHECK ③`src/worker/handlers.ts` 注册 ④`enqueue` 三必填项到位 ⑤把两条既有守卫测试纳入验收（`tests/migrate-transaction.test.ts`、`tests/worker-handlers.test.ts`——「库 CHECK 与 `TaskTypes` 逐条一致」「每个 TaskType 都有 handler」）。

### 10. `tasks.enqueue` 的三个必填参数方案一个都没给

- **哪部分**：`proposal.md:15`「走后台任务异步生成」——没有入队代码
- **后果**：`enqueue` 强制校验 `deduplicationKey`（`src/server/modules/tasks/service.ts:191-193`）、`schedulingGroup`（`:194-196`）、`availableAt`（`:197-199`），任一为空抛 `err.input` → **400**。方案没定义这三者，实现者只能现编，最容易漏的是「重复点导出要不要复用 taskId」——`UNIQUE(task_type, deduplication_key)` 是唯一幂等来源（`:185-188`、`:205-208`）。
- **建议**：`deduplicationKey = export:<userId>:<yyyy-mm-dd>`（同日重复提交自然复用同一 taskId，`created=false` 时直接返回既有 id）；`schedulingGroup` 单独一组，避免导出挤占内容发布任务。

### 11. 300 秒任务死线与「全量导出」在时间上不兼容，且 worker 是串行执行

- **哪部分**：`proposal.md:15` 异步生成、`:63`「不阻塞请求，不影响其他用户」
- **后果**：
  - 单次任务硬上限 `TASK_DEADLINE_MS = 300s`（`src/server/modules/tasks/constants.ts:45`），超时由 `recoverExpired` 回收后**重新执行**（`:24-25`）；可恢复故障最多 3 次、退避 `[30s, 120s]`（`:32-39`）→ 一个全表导出最坏要做 **3 遍全表扫描**。
  - 更关键是执行模型：worker **一轮领最多 4 个，然后逐个 `await execute`**（`src/worker/main.ts:190-205`，全站 `MAX_CONCURRENT_TASKS = 4`，`constants.ts:16`）。导出是长任务，**它会占住一个领取位并阻塞同批所有任务**——即「评测发布、评论审核」与导出排队。方案 §预期 说的「不影响其他用户」**在实现上不成立**。
- **建议**：分批处理（每批 500 行，批间落进度后让出）；把 `progress` 写进 `tasks.finish`（`service.ts:67-68`、`main.ts:93-101` 的用法）；或在 claim 侧限制「全站同时最多 1 个导出」，并在方案里明确「导出会让发布链路的排队时延上升 X」。

### 12. 无限流/无大小上限：内存、磁盘、与健康检查互相引爆

- **哪部分**：`proposal.md:21`（只有一个动作名）、`:53-57`（产物与 TTL），全篇无分页、无上限
- **后果**：
  - 单用户 5000 帖（`body_md ≤ 20000`，`0037_forum.sql:76`）+ 2000 评测修订（`body_markdown` **无长度上限**，`0001_baseline.sql:170`）一次性 `.all()` → worker OOM，而 worker 同时承载发布链路。
  - 产物无上限 → 磁盘打爆。而健康检查会在**剩余 <200MB 时直接判 fail**（`src/server/platform/health.ts:50, 93-96`）→ `/api/health` 503 → Docker HEALTHCHECK 跟着判死。若产物落在 `.data` 同卷（媒体即在 `.data/media`，`src/server/platform/file-store.ts:83, 161`），**写满磁盘会先让站点整体不可用，再顺带打挂 SQLite 写入（WAL 也在该卷）**。
- **建议**：按 id 游标分批 + 流式写（`createGzip`）；设「单次导出行数/字节上限」，超限截断并在产物里写 `truncated: true` + 总数；写入前预检可用空间（预留 ≥2× 预估）。

### 13. §5「TTL 7 天 + 定时清理」在仓库里没有承载者

- **哪部分**：`proposal.md:57` `TTL 7 天，定时清理。`
- **后果**：仓库明确**拒绝长驻定时器**：`rate-limit.ts:182-184` 原文「不开启库的定时清理：它会 setInterval，而我们要进程能被干净关掉」。web 进程没有任何定时能力；全站唯一的周期逻辑挂在 worker 主循环上（`src/worker/main.ts:169-188`，每 `IDLE_SLEEP_MS=10s` 走一次 `runOnce`，排程节流 `DISCOVERY_SCHEDULE_CHECK_MS`）。方案不指定承载者 → 要么没人清理（磁盘涨死），要么有人给 web 加 `setInterval`（违反既有约定、破坏优雅退出）。
- **建议**：清理挂在 worker tick 上（与 `ensureDailyDiscoveryTask` 同款，`src/worker/scheduled.ts`）；写清「产物最长存活 = TTL + 一个 tick 间隔」；清理失败只记日志、不中断主循环。

### 14. 下载接口是越权面最大的路径，方案只有「文件」两个字

- **哪部分**：`proposal.md:12` `GET /api/me/export/:taskId/download → 文件`
- **后果**：`<userId>/<taskId>.json.gz` 是**静态路径**。方案没有要求「`taskId` 必须属于当前 `userId`」「下载时重新认人」「禁止 token-in-URL」。仓库正例是媒体路由：`(attachmentId, variantKey)` 由 `resolveReadable` 判权，**未授权一律 404，不区分「不存在」与「无权限」**（`src/app/media/[attachmentId]/[variantKey]/route.ts:30-49`）。照媒体范式写才对，照本方案写则是「有链接就能拿」。
- **建议**：①按 `(taskId, ctx.userId)` 查询，属主不符统一 404 ②下载重跑 `currentContext()`（仓内会话只走 Cookie，无 token-in-URL 先例）③`content-disposition: attachment` + `cache-control: private, no-store`（媒体路由用的是 `private, max-age=300`，`:45`，导出包不可照抄）。

### 15. 全站零审计，且 `src/` 没有 gzip/流式先例（能力缺口未声明）

- **哪部分**：`proposal.md:1-65` 全篇；`:53-55` 的 `.json.gz` 与 `:12` 的文件下发
- **后果**：
  - **零审计**：导出 = 批量搬走个人数据的高价值动作，仓内已有 `operation_records` + 0042 的不可删改触发器（`migrations/0042_audit_immutable.sql`），方案只字未提。事后无法回答「谁在何时导了多大一份」，账号被盗后的一次性搬空也无迹可查。
  - **能力缺口**：`src/` 里 **没有任何 `zlib`/`gzip`/`createReadStream`/`Readable.toWeb`** 的使用（grep 零命中）——压缩与流式下载都是全新能力。仓库唯一的文件下发先例是媒体路由整体 `new Uint8Array(bytes)`（`route.ts:39-49`），它成立的前提是媒体单文件有限。
- **建议**：导出开始/完成/下载各记一条 `operation_records`（`action_type='data_export'`，记行数/字节/耗时，**不记内容**）；下载用 `createReadStream` + `Readable.toWeb()` 流式返回、`content-length` 取 `fs.stat`；方案写明「压缩用 `node:zlib` 的 `createGzip`，无需新增依赖」。

---

## 三、P2（质量与合规表述）

### 16. §3 导出范围只有 3 张表，却承诺「全部数据」

- **哪部分**：`proposal.md:24-44` vs `:63`「用户可下载自己的全部数据」
- **后果**：库里用户本人的数据至少还散在 11 张表，且都已核实存在：`comments`（`0001_baseline.sql:252`）、`comment_revisions`（`:284`）、`content_drafts`（`:187`）、`review_helpful_votes`（`0012:22`）、`notifications`（`0008:19`）、`user_preferences`（`0030:19`）、`attachments`（`0003_auth_media.sql:45`）、`feedback_tickets`（`0036:19`）、`role_applications`（`0040:48`）、`plugin_ownerships`（`0040:82`）、`plugin_correction_requests`（`0015:20`），以及 `forum_post_replies`（`0037:100`）、`pack_favorites`/`post_favorites`（`0039:67, 80`）。只导 3 张表还对宣称「全部」，是把合规承诺写成空话。
- **建议**：把范围写成「显式清单 + 每项标注是否含本人正文」，另加一节「**明确不导出**及理由」——`user_sessions`（`0001:50`，令牌材料）**绝不能导**，这恰恰是合规评审要看的部分。

### 17. §3 `favorites` 的 `folder_id` 语义被忽略（NULL = 归入默认夹）

- **哪部分**：`proposal.md:38-42` `table:'plugin_favorites', columns:['plugin_id','folder_id','created_at']`
- **后果**：列确实存在（`0019_favorite_folders.sql:26-27` 由 `ALTER TABLE` 追加），但**可空**；设计原文是「历史收藏为 NULL，**读取时归入默认夹**」（`0019:7-8`）。直接导出 `folder_id=null` 而不导 `favorite_folders`（`0019:13-21`）与 `is_default` 标记，用户拿到的收藏**全是没有归属的孤儿行**，无法还原「在哪个夹」。另外新写的两类收藏 `folder_id` 是 `NOT NULL`（`0039:70, 83`），与 `plugin_favorites` **不同构**，忽略这一点就会写出想当然的统一转换。
- **建议**：同时导出 `favorite_folders`（含 `is_default`、`target_kind`），并在产物里把 `folder_id=null` 显式标注为「默认收藏夹（未回填）」。

### 18. 格式与措辞：`schema_version` 有了，但仍不够「可携带」

- **哪部分**：`proposal.md:60`「JSON，带 schema_version 与导出时间戳」
- **后果**：「可携带」的实质是**可移植**，不是「能下载」。方案没给：`schema_version` 的变更承诺（改结构后旧导出变孤儿）、时间字段的单位与时区（全站用毫秒整数，见 `0001:114` 一类 `INTEGER CHECK (>=0)`，对人类不可读）、以及「各表读取时点不一致」的说明——异步 + 重试意味着这份导出没有统一快照时点，事后无法自证。
- **建议**：产物内加 manifest（导出版本、表清单、每表行数、字段说明与类型、时间单位与时区、`snapshot_started_at/finished_at`）+ schema 版本承诺；需要一致性时用一条只读事务包住取数。

---

## 四、方案里「对的」部分（避免推翻重来）

- **异步而非同步返回**，方向正确——5000 帖级别的 `.all()` 同步返回会卡死 Node 单线程（对比 v1 素材 `docs/M13素材-用户数据导出.md:19-31` 的 `SELECT *` + 一次性 `JSON.stringify`）。
- **白名单列**而非 `SELECT *`，方向正确（v1 正是 `SELECT *`）；只是白名单内容本身不含正文（第 4 条）。
- **产物与库分离 + TTL**，思路正确；缺的是落点（第 8 条）与承载者（第 13 条）。

---

## 五、一句话建议

先回答三个问题再动手：**(a) 导出的边界是「我提交过的全部数据」还是「当前公开的内容」**（决定第 6 条）**；(b) 任务在队列里叫什么、谁执行、多久超时**（决定第 9/10/11 条）**；(c) 产物存在哪、谁清理、谁能下载**（决定第 8/13/14 条）。这三件事定下来之前，§2 和 §3 的代码都只能算草稿。
