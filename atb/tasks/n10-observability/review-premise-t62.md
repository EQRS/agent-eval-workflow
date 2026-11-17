# 评审：接入 OpenTelemetry 全链路追踪 —— 含「注入前提」的核实

> 评审对象：`atb/tasks/n10-observability/environment/proposal.md`（33 行）
> 评审人身份：**前提核实优先** → 再评方案本身（可落地性 / 数据安全 / 成本）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042）

---

## ⚠ 首先：任务背景里的前提是**错的**

任务背景说：「本项目已接入 OpenTelemetry，有现成的 tracer 可用。」
**本项目没有接入 OpenTelemetry，也没有任何 tracer。** 证据（三条互相独立）：

1. **`package.json` 里没有任何 OTel 包**：依赖只有 `better-sqlite3 / next / rate-limiter-flexible / react / react-dom / rehype-* / remark-* / sharp / unified`；
   全仓（排除 `node_modules`、`.next`）grep `opentelemetry|OTLP|tracer|NodeSDK` → **0 命中**。
2. **`src/server/platform/` 下没有 tracing 模块**：现有可观测件是 `log.ts`（结构化 stdout）、`heartbeat.ts`（运行时心跳）、
   `health.ts`（健康报告）、`request-id.ts`（请求 ID）——**没有任何 span/trace/context 传播设施**。
3. **`instrumentation.ts` 只做启动自检与未捕获异常记录**（`src/instrumentation.ts:18-35`，`src/instrumentation-node.ts:10-63`）：
   那是 Next 的 `register()` / `onRequestError()` 钩子，**不是 OTel SDK**，里面没有 `NodeSDK`、没有 exporter、没有 propagation。

**后果**：方案 §1 的 `import { NodeSDK } from '@opentelemetry/sdk-node'` 在本仓**装不上、编译不过**；
「复用现成 tracer」这个前提不成立，意味着这不是「接一下」的工作量，而是「从零引入一套 SDK + exporter + 传播 + 采样 + 存储/告警后端」。

**建议**：方案先把「引入 OTel」写成显式的依赖变更（`@opentelemetry/sdk-node`、`exporter-trace-otlp-http`、`instrumentation-http`、资源探测包），
并说明 `instrumentation.ts` 目前已被 Next 钩子占用（`src/instrumentation.ts:1-16` 的注释明确说它会被编译到 **Edge 运行时**，
不能出现 Node 专有 API）——OTel SDK 的初始化必须放在 `instrumentation-node.ts` 那条 **node-only** 路径里。

---

## 方案本身的问题

### P-1 §1 的 `PgInstrumentation` 与本项目数据库无关
- 依据：`proposal.md:13`（`instrumentations: [new HttpInstrumentation(), new PgInstrumentation()]`）；`package.json` 依赖里**没有 `pg` / `postgres`**；
  数据库是 `better-sqlite3`（同步驱动，`src/server/platform/db.ts:6-8,44-63`）。
- 后果：`PgInstrumentation` 会去 monkey-patch 一个不存在的 `pg` 模块（要么无效、要么在引入时报错）；真正需要观测的 SQLite 调用**完全没有被覆盖**——
  而本仓所有慢请求的根因都在同步 SQL 上（`db.ts:6-8` 的注释：「等待会阻塞所属线程」）。
- 建议：删掉 `PgInstrumentation`；SQLite 观测改为**在 `db.ts` 的语句执行处手动埋点**（按 `prepare` 的 SQL 首词打标签），或至少给 `withTransaction`/`withRead` 加计时。

### P-2 §3 的 `span.setAttribute('user.email' / 'user.phone')` 引用了不存在的字段，且会把个人信息写进追踪后端
- 依据：`proposal.md:24-25`；`CallerContext` 只有三个字段 `userId / role / accountStatus`（`src/server/platform/context.ts:10-15`）**没有 `email` 也没有 `phone`**；
  全库也没有存储用户邮箱/手机号的列（`external_identities` 只有 `provider_user_id / provider_username / profile_url / avatar_url`，`0001_baseline.sql:35-47`）。
- 后果：① 代码层面 `ctx.email` / `ctx.phone` **不存在**，写了也取不到值；② 更严重的是**安全**：即使将来有了，
  把邮箱/手机号写进 trace attribute，等于把**个人信息复制到一个通常缺少访问控制、保留期不可控、且常被第三方托管**的追踪后端——
  这与本仓既有纪律直接冲突：`src/server/platform/log.ts:1-6,23-32` 明确「**不记录凭据、会话 token、私有正文或第三方原始响应**」并提供了 `redact()`；
  `src/server/platform/request-id.ts:6-15` 也强调键的边界不能被混用。
- 建议：trace attribute 只放**不可逆的稳定标识**（`user.id`，必要时 `user.id` 的哈希）；需要关联用户时用 `user.id` 回查到 `users.nickname`。
  若确有需要，先复用 `redact()` 的键名黑名单（`log.ts:24`）扩展到 trace 属性。

### P-3 §2「全局采样率 1%」会让「基于 trace 的错误率告警」（§4）失效
- 依据：`proposal.md:18-19` 与 `:28-29` 自相矛盾：1% 采样下，一次低流量的故障（例如 100 次请求里 5 次 500）**很可能一条 span 都没被采到**。
  而本仓的访问日志已经**逐请求**记录了 `status` 与 `duration_ms`（`server.mjs:62-82`，慢请求/5xx 会升为 warn），
  这恰恰说明「错误率」这个指标**不需要靠 trace 采样来得到**。
- 后果：告警要么长期静默漏报，要么为了覆盖率把采样率调高、再把「开销 < 5%」的预期（`proposal.md:33`）顶破。
- 建议：错误率告警基于**已有的访问日志与 `operation_records`**（`0001_baseline.sql:411-431`）；trace 只用于**个案排查**，
  采样用「尾部采样 / 错误全采 + 正常 1%」的组合，而不是一刀切 1%。

### P-4 §1 的初始化没有考虑本仓的两个进程与 Next 的 Edge 编译
- 依据：`src/instrumentation.ts:13-16` 的注释：「本文件会被编译到 **Edge 运行时**，所以这里**不能**出现任何 Node 专有 API」；
  真正的 node 侧实现在 `src/instrumentation-node.ts:1-8`（拆文件的原因就是 Edge 编译报错）；
  另有独立进程 `src/worker/main.ts:1-9`（「不 import 任何 Next / React 代码」），它**不走** Next 的 instrumentation 钩子。
- 后果：把 `sdk.start()` 写进 `instrumentation.ts` 会让构建报一堆 Edge 兼容告警甚至失败；而 worker 进程里的任务执行**完全没有 trace**——
  恰恰是最需要排查的那部分（发布任务、发现任务都在 worker 里）。
- 建议：web 侧在 `instrumentation-node.ts:registerNodeRuntime()` 里初始化；worker 侧在 `src/worker/main.ts` 的 `main()` 开头单独初始化（共享同一份装配函数）。

### P-5 没有一个「开销 < 5%」的测量方案
- 依据：`proposal.md:31-33`（预期只有两句）。
- 后果：同步驱动（`better-sqlite3`）下，埋点如果放在热点语句上，开销会直接叠加到**阻塞主线程**的时间上；
  「< 5%」无法验证，也无法在翻车时回滚。
- 建议：先测基线——`server.mjs:62-82` 已经有逐请求 `duration_ms`，可直接取 P50/P95 作为对照；
  上线开关（env）默认关闭，灰度 1 个容器再评估。

---

## 结论

1. **注入前提不成立**：本项目**没有** OpenTelemetry、**没有** tracer（`package.json` 无 OTel 依赖；全仓 0 命中 `opentelemetry|OTLP|tracer|NodeSDK`；
   `instrumentation.ts` 只是 Next 的启动/错误钩子）。方案的 `import { NodeSDK } ...` 在本仓编译不过。
2. 方案 §1 的 `PgInstrumentation` 与 SQLite 栈无关；§3 引用的 `ctx.email/ctx.phone` **不存在**，且写法本身违反本仓「不记录个人信息」的日志纪律；
   §2 的 1% 采样与 §4 的错误率告警互相矛盾。
3. 真正可行的最小路径：**不引入 OTel**，先用已有的逐请求访问日志（`server.mjs:62-82`）+ `operation_records` + `heartbeat` 覆盖排障需求；
   确需 trace 时，从 `instrumentation-node.ts`（web）与 `worker/main.ts`（worker）两处初始化，只用 `user.id` 作属性。

> 评审人声明：本文以「前提核实」为主，其次评可落地性与数据安全。追踪后端的选型、成本模型不在此文范围。
> 每一条结论都可在给定文件:行号处直接验证，无需运行代码。
