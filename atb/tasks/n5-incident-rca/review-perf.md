# 评审 · 性能工程师（日志量 / 采样 / 热路径开销）— n5-incident-rca 方案

评审对象：`tasks/n5-incident-rca/environment/proposal.md`（37 行）
口径：只看性能面（日志量、采样、热路径开销），不做安全/产品口径判断。

## 摘要（前 5 条最重要）

1. 【严重】§1 删掉 stack / error_message 等于把 500 的唯一根因线索删掉，而"日志量太大"这个前提在代码里站不住：全仓 error/warn 站点仅 21 处，且全在失败路径；真正的日志量主体是 server.mjs 的每请求访问日志。— 依据：`src/server/platform/log.ts:12-21,34-38`、`src/server/platform/errors-http.ts:114-128`、`src/instrumentation-node.ts:50-63`、`server.mjs:39-43,62-82`、`docker/compose.yaml:70-74` — 建议：保留 error_message + 截断后的 stack（前 N 帧），脱敏交给 `log.redact`（log.ts:24-32）；省量的手段放到访问日志采样与错误签名聚合，不动错误字段。
2. 【严重】提案没有"日志采样/去重"这一层，现网也没有：`emit()` 无采样、无按签名聚合、无速率上限。一次 500 风暴（同一条堆栈每请求一行）会在 json-file 10m×3 上限内把保留窗口刷掉，导致"想查时日志已经被自己冲掉"。— 依据：`src/server/platform/log.ts:12-21`、`docker/compose.yaml:70-74`、`docker/daemon.json:1-8` — 建议：按 `(错误签名 = error_name + 首帧 + route_path)` 做"每窗口首条全量 + 后续计数"，窗口结束补一行汇总；上限必须显式（如每签名 5 条/分钟，每进程 200 行/分钟）。
3. 【严重】§3 的"500 > 100/分钟告警"是绝对速率、无分母无基线，且**当前没有任何地方产出这个数**：仓内无 metrics/counter/告警通道，访问日志是逐请求行、错误日志在 stderr，只能靠日志聚合器算；单路由风暴（100/min 可能只来自一个路径）与全站慢涨（100/min 可能是 10 倍基线）无法区分。— 依据：`server.mjs:67-81`、`src/server/platform/log.ts:19`、`src/app/api/health/route.ts:30-58`（健康端点明确"不做业务统计"）— 建议：改成分式判据 = 500 数 / 总请求数（5xx 率）+ 按 path 分组 Top-N，速率阈值按 (全局 5xx 率 >1% 持续 5min) 或 (单路径 >20/min 且占比 >50%) 触发。
4. 【高】热路径开销：访问日志在 `res.on('finish')` 里每请求一次 `JSON.stringify` + 同步 `process.stdout.write`（管道/文件），`startedAt = Date.now()` 每请求两次取时；无采样、无慢请求阈值外的分级保留。500 风暴时 stderr 写入量翻倍（`status>=500||slow` → warn → stderr，而 slow 默认 1000ms 且 `DSH_REVIEW_SLOW_REQUEST_MS` 未进 .env.example）。— 依据：`server.mjs:50-82`、`server.mjs:67`、`docker/.env.example:1-66`（无 slow 阈值项）— 建议：给访问日志加确定性采样（成功 200 全留/1xx-3xx 抽 1/10、5xx 全留）或至少把慢请求阈值纳入配置并写进 .env.example；同步写在大流量下要考虑 backpressure（stdout 返回 false 时的计数而不是每行检查）。
5. 【高】提案 §4"记录 500 错误总数"没有落点：仓内无指标端点、无计数器表、health 端点刻意不统计。要真正落地得靠日志聚合（从 json-file/journal 侧算），这恰恰与 §1"删字段"相反——字段越少，聚合维度越少（无 route/无 error_name 就只能按 message 分桶成一条）。— 依据：`src/app/api/health/route.ts:1-15`、`src/server/platform/health.ts:1-11`、`src/server/platform/log.ts:8-10`（LogFields 只允许标量）— 建议：先定义"错误签名"这个聚合键并保证它进日志，再谈总数与告警；签名至少要 `error_name + 首帧 + route_path + status`。

## 逐条问题

### P1【严重】§1 的前提（"日志量太大")与代码现状不符，删字段是拿最贵的字段换最小的收益
- `log.` 调用点合计 21 处（grep `log\.(error|warn)\(` → 21），全部是失败/降级路径；没有一处位于每请求/每轮循环的热路径上。错误日志的体量由 500 发生次数决定，不由"带不带 stack"决定（stack 每行 ~1-3 KB，但只在 500 时出现一次）。
- 真正的体量在访问日志：`server.mjs:62-82` 每请求 1 行 JSON（`http_request`），且 500 请求会**同时**产生访问日志 1 行 + 业务错误日志若干行，属 n+1。
- 结论：删 stack 把定位能力降为零，却只减少了"发生错误时"的那一小部分字节。**该删的是成功请求的访问日志，而不是失败请求的堆栈。**

### P2【严重】提案给的 `log.error({...})` 形态与现有签名不兼容，且一旦改动会让所有 21 个站点静默失字段
- 现状签名：`log.error(message: string, fields?: LogFields)`（`src/server/platform/log.ts:34-38`）。提案写的是单对象调用；若按提案改签名，`errors-http.ts:120`、`instrumentation-node.ts:50` 等站点会变成 `message = undefined`，字段整体丢失——这是"日志量下降"的最容易误判来源（量下降了但原因是字段没了）。
- `LogFields` 只接受标量（`log.ts:8-10`），`stack` 也只是一个 string 字段，并没有嵌套开销问题。

### P3【严重】无采样/去重/限流，是本次"日志量失控"的真实机制
- `emit()`（`log.ts:12-21`）：每个调用无条件 JSON.stringify + 一次同步写。没有"同一条错误每分钟只写几次"的机制。
- 可预期的风暴源：GitHub 上游故障（`src/server/adapters/catalog/github-probe.ts:260-265,283-285,395-400,456-461` 全部把 403/429/非 2xx 映射为 provider_unavailable → 503；`src/server/modules/catalog/discovery.ts:226-231,291-296` 在批量发现时逐候选 log.error 后 rethrow）。批量发现会逐个候选打错误，一轮 12 候选 × 每次 3 请求的超时路径，能产出成百行/轮的 stderr。
- 落地建议：在 `emit()` 内加"签名窗口计数"——签名 = `level + message + error_name + stack 首帧 + route_path`，每签名每 60s 首条全量 + 期间只计数，窗口结束时补 `{repeated: N}`。这样既保住了堆栈（首条有），又把风暴期日志压到常数级。

### P4【严重】§3 告警阈值缺分母与基线，且缺"从哪读"的落点
- `server.mjs:67`：`status >= 500 || slow` 时才写 warn（stderr），其余 info（stdout）。500 的"数"就是日志行数，没有单独计数器。
- 仓内不存在 metrics/alert 通路（grep `metrics|counter|prometheus|sentry|alert` 在 src 下零命中，`src/server/modules/notifications/index.ts:12` 明确"要加邮件/webhook 时再补"）。所以 100/min 只能由日志聚合器算——而提案没写这一层。
- 100/min 绝对值的失效场景：(a) 单路径风暴，100/min 来自一个接口，远超引擎正常水平，但全站率可能仍 <1%；(b) 全站放量后基线从 2/min 抬到 90/min，永不告警。建议加分母：`5xx_rate = 5xx / total`（server.mjs 已有每请求行，可直接算），并按 path 分组。

### P5【高】访问日志热路径开销与可配置性缺口
- `server.mjs:50-82`：`Date.now()` 每请求一次（另 `durationMs` 再算一次），`JSON.stringify` 每请求一次，`process.stdout.write` 同步。单请求成本不大（微秒级），但 500 风暴时行数放大几倍，且 stderr 与 stdout 不同的落盘队列不保证对齐。
- `DSH_REVIEW_SLOW_REQUEST_MS` 在 `server.mjs:18,31` 有默认 1000，但 `docker/.env.example` 里没有该项（`.env.example:1-66` 逐项核对），也没进 compose 的 environment（`docker/compose.yaml:50-62`）。建议补上并在告警里把 slow 与 5xx 分开统计（现在的 `warn` 把两者合并，会让"慢"淹没"错"）。
- 建议采样策略：502/504 与 5xx 全留；2xx/3xx 抽 10%；静态与 health 已静音（`server.mjs:35-37` 已做，好）。注意采样必须**确定性**（按 request_id 哈希）而不是随机，否则同一次报障无法复现。

### P6【高】worker 侧另有隐性热写入，与"日志量"混淆
- `src/worker/main.ts:273` `onTick: () => heartbeat.beatNow()` 在每次 `runOnce`（空闲 10s 一次，`constants.ts:48`）写一次 `runtime_heartbeats`，另有 `startHeartbeat` 5s 一次（`heartbeat.ts:26,148`）。这是**写库**不是写日志，但在 synchronous=FULL（`db.ts:54`）下单次 fsync 有成本。提案若把"日志量"扩展到"IO 量"，应把这条与日志分开计。
- `src/worker/main.ts:154-158` 每个任务结束一条 `log.info('任务结束')`，属正常量，不是问题；建议保留（它是"500 是不是 worker 引起的"的关键对照）。

### P7【中】§2 的排查步骤缺"日志能不能被检索"这一前提
- 现在日志只有 json-file（10m×3，`compose.yaml:70-74`）= 单容器最多 30MB ≈ 数分钟到数小时的量（按 500 条/分钟 × 1KB 算，1 分钟就能吃掉 1MB）。`deploy/*.service` 走 journal（`dsh-review-web.service:29-31`），受 journald 默认 RateLimitBurst 500 条/30s 限制——风暴时会被 journald 直接丢行。提案 §2"看日志找路径分布"在这个前提下不可执行。
- 建议：§2 补一步"先把错误日志落到可聚合处（journald 的持久化 + 提高 RateLimitBurst，或 sidecar 收集）"，否则流程只在低流量时成立。

### P8【中】缺少"保留成本"这个显式约束
- 提案的"预期：日志量下降"没有给数量级目标，也就没法验收。建议写成可测判据：`500 风暴期间（>100 5xx/min）单进程 stderr ≤ 200 行/分钟，且保留窗口 ≥ 6 小时`。这样 §1 的字段取舍才有依据（删字段对这两个数字贡献极小，聚合与采样贡献极大）。

## 不适用条件（本条评审的边界）
- 不评价"是否泄露内部实现"这一安全口径（stack 只进服务端日志、不回传响应，已在 `errors-http.ts:9` 约定；脱敏已有 `log.redact`）。
- 不评价告警通道选型（邮件/webhook/Prometheus），只评价"阈值口径与数据来源是否成立"。
- 不覆盖 worker 的任务并发与租约参数调优，只指出与日志/IO 量相关的两处。
