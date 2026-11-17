# 评审报告：n5-incident-rca《线上 500 错误排查流程》方案

**对象**：`tasks/n5-incident-rca/environment/proposal.md`（35 行）
**结论**：**不通过（Reject）**。方案的两个前提在本仓都不成立——"不记堆栈"会直接废掉仓库里唯一能定位 500 的证据链，"100 次/分钟告警"在 SQLite 与日志轮转的部署形态下既打不到也不准。

---

## P0（阻断发布，会直接让评审者付出错误的代价）

### P0-1 §1「不记录错误堆栈和具体原因」把仓库现成的错误链路反过来了
- **位置**：proposal.md:19（`不记录错误堆栈和具体原因`）+ proposal.md:11-16（示例 `log.error({...})`）。
- **事实**：本仓的错误日志**专门**记 `error_message` 与 `stack`：
  - `src/server/platform/errors-http.ts:120-127`：`unhandled_error` 记录 `request_id/code/error_name/error_message/stack/context`，并在同文件 :111-112 明确写着"堆栈走日志而非响应体，因此用户拿到的只有可展示说明与 request_id"；
  - `src/instrumentation-node.ts:50-63`：页面级未捕获异常记 `path/route_path/digest/error_message/stack`；
  - 文档侧的排查链也在用这套：`docs/项目规划/部署-README.md:190`（"未捕获异常一行含 path/digest/stack"）、:205-211（按 digest grep 日志定位）。
- **后果**：照方案实施等于给 `stack` 字段"降级"。线上 500 发生时，用户报一个 `request_id`/`digest`，日志里只剩 `message: 'internal error'`——**没有任何信息可以定位**。排查步骤 §2 第 2 步"按路径找到对应的代码"因此恒不可执行。方案自己的目标"快速定位问题"被 §1 直接抵消。
- **安全前提也站不住**：本仓的泄露边界是**响应体白名单**（`errors-http.ts:49-60` 的 `toPublicError` 只回 `code/message/fields/context`），日志本来就只进服务端 stdout，不对外。"不记堆栈避免泄露内部实现"把响应体纪律误套到服务端日志上。
- **建议**：保留并规范化堆栈（`stack` + `error_name/error_message` + `request_id` + `route_path`），只约束"不准进响应体"；如担心第三方返回体/用户正文进日志，用**字段级脱敏**（扩展 `log.ts:24` 的 `SENSITIVE_KEYS`——目前只按键名匹配，管不住 `error_message` 里的内容）而不是删掉整条链。

### P0-2 §1 示例日志与仓里的 log 契约不兼容，且只覆盖了"半条命"
- **位置**：proposal.md:10-17。
- **事实一（API 不匹配）**：`src/server/platform/log.ts:34-38` 的签名是 `log.error(message, fields)`，示例写成 `log.error({ status, path, userId, message })`——第一个参数会被当成 message 输出一个对象，`userId` 也过不了 `redact()` 之外的任何规约。照抄即得到形如 `{"ts":...,"level":"error","message":{"status":500,...}}` 的无用行。
- **事实二（覆盖不全）**：500 不止一条来源。`src/worker/main.ts:128-133` 把处理器异常吞成 `errorCode:'handler_threw', errorSummary: e.message`，只写回队列、**不落 `log.error`**（对比同文件 :154 成功路径有 `log.info`）；`src/app/api/pack-releases/[releaseId]/submit/route.ts:20-22` 用的是 `console.error` 而非统一 `log`。方案只规定 HTTP 入口那一条，worker 侧失败与影子异步失败仍然不可见。
- **建议**：示例改成 `log.error('unhandled_error', { request_id, code, error_name, error_message, stack, route_path })`；把 worker 的 `handler_threw` 补齐成 `log.error`（含 stack），把 `console.error` 收口到 `log`；在方案里点名"500 的三个来源：路由未捕获、页面渲染未捕获（instrumentation）、worker 任务失败"。

### P0-3 §3 告警阈值「100/分钟」在这个部署形态下既触发不了也不可信
- **位置**：proposal.md:28。
- **事实**：`server.mjs:62-82` 每请求一行 `status`，但日志落在容器 `json-file` 且 `max-size: 10m / max-file: 3`（`docker/compose.yaml:70-74` web、:95-99 worker）。突发 500 时日志**先被轮转**，事后想统计"过去一分钟多少 500"已经缺数据。
- **后果**：① 站点体量下 500/分钟几乎到不了，告警形同虚设（一个接口 100% 报错、QPS 5 也只有 5 次/分钟）；② 真到 100/分钟的雪崩时，日志已被覆盖 20-50 秒前的证据——**排查最需要现场的时刻恰恰是现场被删的时刻**。这与 `docs/项目规划/部署-README.md:181`（日志有上限）同源，但方案没有处理它。
- **建议**：改成**比例 + 绝对**双阈值（如 5xx 占比 ≥2% 持续 5 分钟，或 ≥10 次/分钟），并在**应用内**用计数器（`/api/health` 同源的进程内滑动窗口，或直接落 SQLite 一张 `http_errors_by_minute` 表）产出，而不是事后 grep 日志；同时把"日志 30 分钟内会被轮转"写进方案的排查步骤（调大 `max-file`，或把 error 级单独打到另一个 sink）。

### P0-4 §3/§4 没有"谁来看"：本仓根本没有告警通道
- **位置**：proposal.md:27-31。
- **事实**：全仓无 Prometheus/Sentry/OTel 依赖（`package.json:25-38` 无任何一个），deploy/CI 也无外部告警配置；现有"告警"只有容器健康检查——web 的 `/api/health`（`src/app/api/health/route.ts:52-58`，且按设计**不含** worker 心跳，见 :13-14）与 worker 的 `scripts/worker-healthcheck.mjs`（`docker/compose.yaml:103-108`）。两者都**看不见 500**，也**不会通知任何人**（`restart: unless-stopped` 只在容器退出时生效）。
- **后果**：方案承诺的"超过 100/分钟告警"没有任何执行载体；上线即沉默。而且这条阈值挂在**进程外**的指标上，与仓里"健康检查走状态码"的既有约定不一致。
- **建议**：明确落地物二选一——(a) 在 web 进程内做滑动窗口 5xx 计数并暴露 `GET /api/health` 已有信封之外的字段（或新增受 token 保护的 `/api/metrics`，复用 `x-health-token` 的 `isDetailAllowed` 模式，`health/route.ts:23-28`），由外部采集；(b) 至少把告警接到现有 healthcheck 语义上（进入 fail → 503 → 编排/巡检脚本发现问题）。方案里必须写"告警接收方是哪个系统"。

### P0-5 §4「只记 500 总数」指标维度不足，无法回答"哪个路径在坏"
- **位置**：proposal.md:30-31。
- **事实**：`server.mjs:67-81` 已有的访问日志字段是 `request_id/client_ip/method/path/status/duration_ms/slow`——**path 与 method 本来就有**。方案只保留"总数"，等于把已有的最有用维度丢掉。
- **后果**：§2 第 1 步"看日志，找出 500 的路径分布"依赖的正是 path 维度；方案自己的排查步骤 1 与自己的指标定义 4 互相矛盾（日志有 path、指标没有），执行者会各干一套。
- **建议**：指标口径改为 `status × route_path × error_code`（路由模板而非原始 URL——`server.mjs:52` 的 `path` 是原始 URL，含 `/api/reviews/<id>` 之类高基数 id，直接按它聚合会爆炸）。注意 `request_error` 里已有 `route_path`（`instrumentation-node.ts:54`），可复用它做归并键。

---

## P1（会显著拉长 MTTR / 引入新风险）

### P1-6 §2 排查步骤是"人肉四步"，缺定位键与复现条件；本仓已有更完整的链
- **位置**：proposal.md:21-25。
- **事实**：四步里没有 `request_id`/`digest`/`route_path`/`error_code` 任何一个键。而仓里这条链是**已经建成并实测过**的：`docs/项目规划/部署-README.md:203-215`（digest → grep → `status:5` 看影响面 → `diagnose.mjs`）、`README-切片说明.md:104`（用户看到的编号 = 日志里的 digest）。
- **后果**：方案从零重写一套更弱的流程，且第 3 步"本地复现"对线上偶发问题经常不成立（依赖上游/并发/数据状态），没有兜底路径就会卡死。
- **建议**：直接承接现有链路（编号 → `request_id` → 访问日志 + `unhandled_error` → `route_path`），把"复现不了怎么办"写清楚：线上 SQLite 可只读快照（`db.ts:20-23` 支持 `readonly`）+ `scripts/diagnose.mjs`，而不是"本地复现"。

### P1-7 §1 指定记录 `userId` 与 path 原文，和本仓的隐私/日志纪律冲突
- **位置**：proposal.md:13-14（`path: req.url`、`userId: ctx.userId`）。
- **事实**：`log.ts:1-6` 明令"不记录凭据、会话 token、私有正文"；`request-id.ts:30-38` 专门防"日志注入"，说明本仓对"什么东西可以进日志"是有审过的；`docs/API端点与路径.md:46` 的口径是"堆栈只进服务端日志"。
- **后果**：`req.url` 是原始 URL（含 query），会把 `?returnTo=...`、搜索词、甚至 OAuth `code`（`src/app/api/auth/github/callback`）原样写进日志；`userId` 是稳定标识，进日志后日志一旦外泄就是身份关联。方案自己却在 §预期（:34）写"不泄露内部信息"——目标与手段冲突。
- **建议**：记 `route_path`（模板）而非 `req.url`；`userId` 改为 `user_hash`（`digest.ts` 已有哈希设施可复用）或干脆不记、只记 `anonymous/authenticated`。

### P1-8 §2 没有"500 之外"的分流，会把预期业务错误当故障处理
- **位置**：proposal.md:8-9、21-25。
- **事实**：本仓把"业务错误"与"未知异常"分得很清：`errors-http.ts:114-119` 里 `isDomainError` 的业务错误只写 `warn('domain_error')`，只有未知异常才写 `error('unhandled_error')` 并带回 500。而 `history` 里多条真实故障的形态恰恰是**裸 `Error` 被折成 500**：`README-切片说明.md:285`（`assertCanWrite` 抛裸 Error）、`:292`（路由解构错字段）、`模块签名契约.md:378`（`CHECK` 拒绝冒出裸 SqliteError）。
- **后果**：若不区分，一是告警会被正常业务拒绝（429/401 之类不会 500，但**误折成 500 的 bug** 混合其中）淹没，二是排查时会把"某处抛了裸 Error"当成基础设施问题去查。
- **建议**：告警只统计 `message='unhandled_error'/'request_error'` 这一类；并在方案里加一条**反向治理**：定期扫 `internal` 5xx，把"本该是 4xx 的裸 Error"逐条改成 `DomainError`（这正是本仓历史上反复踩的坑）。

### P1-9 预算/轮转未讨论，与"偶发 500"的时间窗冲突
- **位置**：proposal.md:4（"偶发"）与 :27-31。
- **事实**：`docker/compose.yaml:70-74` 的 `10m×3` 在高峰期（访问日志每请求一行，见 `server.mjs:62`）保留窗口可能只有几十分钟。
- **后果**："偶发"若发生在夜里，第二天上班时日志已轮转，§2 第 1 步直接无数据可看。
- **建议**：把"错误级日志单独落盘/单独 sink、独立轮转策略"写进方案（error 与 access 分离是低成本改动），或明确要求 `DSH_REVIEW_LOG_MAX_FILE` 可配并给出建议值。

### P1-10 现有错误分层没被利用，方案缺少"错误码分布"这一关键维度
- **位置**：proposal.md:30（"记录 500 错误总数"）。
- **事实**：`errors-http.ts:28-43` 有完整的 `ErrorCode → status` 映射，`unhandled_error` 里已带 `code` 字段；`docs/API端点与路径.md:46` 也有对外口径。
- **后果**：只数总数，无法回答"是哪个错误码在涨"（internal / provider_unavailable 503 / target_unavailable 404 混在一起会被误判）；而 `provider_unavailable` 是 **503 不是 500**（`errors-http.ts:40-41`），只盯 500 会漏掉整类上游故障。
- **建议**：指标改成"按 `code × route_path` 的 5xx 分布"，并把 503（上游故障）纳入告警面——本仓大量 500 的真实根因在 GitHub 上游（`github-probe.ts:66` 15s 超时、`discovery.ts:173-320` 多处上游失败路径）。

---

## P2（可改进）

### P2-11 请求体没有统一的体积上限 → `request.json()` 失败的样子可能根本不是 500
- **位置**：proposal.md:8-9（假定 500 都走同一条日志）；关联 `src/app/api/_lib/context.ts:60-66`。
- **事实**：`readJson` 直接 `request.json()`，全仓只有 `src/app/api/attachments/.../file/route.ts:19-45` 自己做了 `maxBytes` 限制；其余 60+ 处（`src/app/api/_lib/context.ts:60`、`src/app/api/plugins/route.ts:53`、`src/server/modules/imports/index.ts` 清单体等）无上限。
- **后果**：超大 body 的表现是天真的 500 或 socket 中断，会是"偶发 500"的**真实来源之一**，但方案按"500 都记日志"处理，会看到大量无栈的空壳行。
- **建议**：排查清单里加一条"先看是否有统一 body 上限"的核查项；并作为 P1 修复候选：在 `readJson` 处加 `content-length` + 流式累计上限（复用 `file/route.ts:19-45` 的 `readBodyWithinLimit` 写法）。

### P2-12 长事务 + 同步驱动会表现为"500 与变慢同时发生"，方案没覆盖
- **位置**：proposal.md:21-25（只按"500"排查）。
- **事实**：`better-sqlite3` 是同步驱动、`busy_timeout = 5000`（`db.ts:56`），`docs/实测记录-M6多专业会诊.md:79` 已记录"高并发下必然 SQLITE_BUSY"；且 `src/server/modules/imports/index.ts:424`（`for (const row of targets)` 默认 30 条、每条串行打 GitHub，`discovery.ts:42-43` `maxPlugins:30/maxRequests:400`）走的是同步 HTTP 路由 `src/app/api/admin/imports/[batchId]/run/route.ts:30-32`，单次请求可能跑几分钟——期间整进程事件循环被 GitHub 往返与同步 DB 交替占用。
- **后果**：管理员跑一次导入/后台巡检时，站点会出现"一批 500 + 大量 slow 请求"。这类故障的根因不是某个处理器抛错，而是**同一进程里的网络等待**；方案的"看 500 → 找代码"路径会指向所有 handler，唯独不指向导入。
- **建议**：诊断手册加"500 突发时先看是否有人同时在跑批量导入/自动发现"；中长期把长任务挪到 worker（`src/worker/main.ts` 已在跑队列）——这也正好解释为什么 `server.mjs:31` 有 `DSH_REVIEW_SLOW_REQUEST_MS=1000` 的 slow 标记，方案应把它纳入告警。

### P2-13 "统一记录"没有形成单一入口，迟早漂移
- **位置**：proposal.md:8-9。
- **事实**：仓里已有三个日志入口且口径不一：`errors-http.ts:117/120`（结构化 log）、`instrumentation-node.ts:50`（结构化 log）、`server.mjs:39-43`（自写 `writeLine`，**绕过 `log.ts` 的 redact**）、以及 `session-cookie.ts:62` / `pack-releases/[releaseId]/submit/route.ts:21` 的裸 `console`。
- **后果**：方案说"所有 500 错误统一记录"，但没有任何机制保证——尤其 `server.mjs` 那条路径不进 `redact()`。
- **建议**：方案里把"统一"具体化为"500 一律经由 `errorResponse`/`logUnknownError` 出口"，并在 pre-commit 或 `npm run typecheck` 同层加一条简单门禁（如禁止业务代码直接用 `console.`，与 `check:wording` 同类）。

---

## 风险汇总（按"上线后会怎样"排）

| # | 风险 | 触发条件 | 影响 |
|---|---|---|---|
| 1 | 无栈 500，等于不可诊断 | 任何线上 500 | MTTR 从"照着 digest grep"退化到"猜" |
| 2 | 告警永不触发 | 站点 QPS 低 | 故障靠用户投诉发现 |
| 3 | 告警真触发时证据已被轮转 | 日志 10m×3 | 最需要现场时现场丢失 |
| 4 | 路径/上游维度丢失 | 只数总数 | 无法区分自身 bug 与 GitHub 上游抖动 |
| 5 | 隐私边界被 `req.url`+`userId` 打开 | 含 query 的请求 | 日志外泄即身份/令牌泄露 |

## 建议的最小修改（可落地版）

1. **删掉** proposal.md:19 的"不记录堆栈"，改为"堆栈只进服务端日志、绝不进响应体"，并引用 `errors-http.ts:9` 的既有硬规则。
2. 日志示例改为仓内真实签名：`log.error('unhandled_error', { request_id, route_path, method, status, code, error_name, error_message, stack })`。
3. 保留 path 维度但用 `route_path`（`instrumentation-node.ts:54` 已有），不记原始 URL 与明文 `userId`。
4. 告警改为"5xx 占比 + 绝对次数"双阈值，载体写明（进程内滑窗 + `/api/metrics` 或 health 扩展），并把 503 纳入。
5. 排查步骤承接到 `docs/项目规划/部署-README.md:203-215` 的既有链路，并补"复现不了"分支（只读快照 + `diagnose.mjs`）。
6. 补 worker 侧：`src/worker/main.ts:128-133` 的 `handler_threw` 落 `log.error`（含 stack）。
