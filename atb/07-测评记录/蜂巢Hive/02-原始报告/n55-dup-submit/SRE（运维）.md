# n55-dup-submit · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0022
> 模型 deepseek-chat · 3760 字 · 22 次工具调用

---

证据已齐备。`UNIQUE (submitted_by, request_key)` 位于 `0001_baseline.sql:325`，这就是该方案本应使用的机制。

## SRE 评审意见：表单防重复提交方案

**总体判断：不批准。** 该方案的两个机制在运维侧都不可靠：**前端的措施会在故障时失效；后端的措施在多进程部署下会失效**。而且仓库里已经有正确做法（`requestKey` 幂等）却未被采用。

---

### 1. 用进程内 `Set` 做后端去重，在部署形态下必然失效（P0，阻塞）

**① 问题位置**：方案 §2「后端加锁」`const inFlight = new Set<string>()`。

**② 后果**：

- `src/server/platform/db.ts:76` 明确写道「进程级单例连接（**Web 与 worker 各自一个进程**）」；`src/worker/main.ts` 也确为独立进程入口。因此 `Set` 是进程内的，**两个 Web 实例之间、Web 与 worker 之间完全互不可见**。
- 双击请求经负载均衡可能落到不同实例，两次都能通过检查，**仍会重复创建**——而这正是要解决的场景。
- 更严重的是：`MAX_CONCURRENT_TASKS = 4`（`src/server/modules/tasks/constants.ts:18`）说明同一进程内有并发，但 `Set` 没有原子性；`has()` 与 `add()` 之间存在竞态，异步间隙内第二个请求仍会穿过。
- **清理风险**：`finally { inFlight.delete(userId) }` 在进程被 `SIGTERM`/OOM/发布重启时不会执行。若将来改为持久结构或增加人工解封，必然出现「**用户被永久锁死**」的工单，且现场无日志、无审计、无法定位是哪次请求留下的（`Set` 不落盘、不记录）。
- 键只用 `userId`（方案原文 `inFlight.has(userId)`），意味着**用户在提交 A 表单期间，任何其它写入都被拒绝并返回 409**。用户会看到「请勿重复提交」但实际点的不是同一个表单，这类误报会被当作 bug 上报运维。

**③ 建议**：放弃内存 `Set`。改用仓库既有的数据库级幂等：`content_submissions` 已有 `UNIQUE (submitted_by, request_key)`（`migrations/0001_baseline.sql:325`），评论模块已实现 `CreateCommentResult.created` 语义（`src/server/modules/comments/index.ts:50-58`，注释即写明「false = 命中同作者同 requestKey 的幂等重放」）。**让唯一约束在数据库层拒绝重复，而不是在进程内存里猜**——这是 `0008_notifications.sql:12-14` 已经确立的纪律：「做成数据库约束，而不是靠应用层先查后插——并发下也不会插出两条」。

### 2. 没有前端重试或幂等键的兜底，用户重试会造出脏数据（P0）

**① 问题位置**：方案 §1 前端代码 `await submit(); setSubmitting(false);`——**没有 `try/finally`**。

**② 后果**：

- `submit()` 抛错（网络抖动、`busy_timeout` 超时）时 `setSubmitting(false)` 永不执行，`src/server/platform/db.ts:40` 的 `busy_timeout` 默认 5000ms，WAL + `synchronous = FULL` 下慢盘完全可能触发。**按钮永久卡在 disabled，用户只能刷新或换个标签页重来**——刷新后前端状态清零，再提交一次，若第一次其实已成功落库，就产生了**重复数据**。前端禁用反而把故障从「可重试」变成「必重复」。
- 方案全篇没有传递任何**幂等键**。这意味着「前端禁用」是唯一防线；用户关掉 JS、直接调接口或用两台设备，防线归零。
- 方案「预期」只写了「不会重复提交」，**没有定义「已成功但客户端超时」这一情形**——这正是重复提交的真正来源。

**③ 建议**：`try/catch/finally` 复原按钮；提交时生成一次性 `request_key`（UUID）随请求发送，**并在失败重试时复用同一个 key**，让服务端 `UNIQUE (submitted_by, request_key)` 把重试折叠成同一笔。前端禁用只当**体验优化**，不可当正确性保证——这一条要明确写进方案。

### 3. 与现有频控机制冲突，会产生误导性 409 和额度计算错误（P1）

**① 问题位置**：方案 §2 抛 `err.conflict('请勿重复提交')`。

**② 后果**：

- `src/server/platform/rate-limit.ts` 的 `RATE_LIMIT_RULES` 已把 `comment_change` 等写入合并计数，且注释明确要求「**相同已成功请求键的重放返回原结果，不另算一次提交**」，重放时由路由调 `ticket.refund()` 退还额度。方案新加一道 409 会**在频控与业务之间插进第二套「拒绝」语义**：命中重放的用户收到 409 而不是原结果，**额度已经扣了却没退还**，用户被白扣一次配额。
- 运维视角最痛的是**错误码语义污染**：409 现在无法区分「真的重复」与「进程内残留锁」，值班同学看告警时无法判断该不该介入。

**③ 建议**：不要新增 409 路径。复用已有的 `created: false` 幂等返回（`comments/index.ts:50-58`），让重放**返回原结果而非报错**，并沿用 `refundQuietly`（`rate-limit.ts` 尾部）退还额度。

### 4. 缺少可观测性，故障时无法排障（P1）

**① 问题位置**：方案 §2 只写抛错，全文**无一条日志、指标或审计**。

**② 后果**：线上出现「用户说点了没反应/锁死了」，运维手上没有任何依据：不知道命中过多少次、是哪个用户、是重复还是误锁。仓库本身有 `log`（`src/server/platform/log.ts`）与 `withTransaction`（`src/server/platform/tx.ts`）基础设施，方案未使用。对比 `0006_rate_limiter.sql` 的注释——它连「被拒请求也会加计数」「超额刷不会把封禁越刷越长」这种反常行为都写进文档，因为**运维必须知道系统的真实行为边界**。本方案一个行为细节都没交代。

**③ 建议**：至少记录命中幂等重放的事件（用户、动作、request_key），并给出可核对的指标口径，便于值班判断是「正常去重」还是「异常锁定」。

### 5. `err.conflict` 未处理事务边界，与「短事务」纪律不符（P2）

**① 问题位置**：方案把 `handle()` 包在锁内。

**② 后果**：`src/server/platform/db.ts:11-13` 与 `src/server/platform/tx.ts:5-8` 要求「业务事务保持短小，不在事务内做网络请求、图片处理或 Markdown 解析」。若 `handle()` 内包含外部调用（评论检查任务、内容筛查 `src/worker/content-screen.ts` 存在外部服务交互），**持锁时间 = 外部服务响应时间**。同用户后续请求全被 409，且锁在 `finally` 前不释放，外部服务慢时直接放大成可用性问题。

**③ 建议**：幂等判定必须在**事务内部、且在任何外部调用之前**完成（唯一约束天然做到这点）；外部副作用留在事务之外。

---

### 结论

`src/worker/` 与 Web 是**两个进程**，`MAX_CONCURRENT_TASKS = 4` 表明进程内也有并发——**内存锁的适用范围为零**。同时 `request_key` 幂等链路在 `0001_baseline.sql:325` 与 `comments/index.ts:50-58` 已完整落地，方案却另造一套进程内 `Set`，属于**重复建设且更弱的实现**。

建议方案重写为：**前端 `try/finally` + 稳定 `request_key` 复用；后端靠 `UNIQUE (submitted_by, request_key)` 幂等并返回原结果；重放退还频控额度；补一条幂等命中日志。** 改完后我再从发布/回滚/监控角度过一遍。