# m10-idempotent-queue · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_题库全测_验证单元_0003
> 模型 deepseek-chat · 8044 字 · 20 次工具调用

---

评审完成。以下只从 SRE（运维）视角，聚焦**上线后会不会半夜叫醒我**的问题。

---

## 一、致命：手动触发「幂等」是假幂等，会把人工重跑永久锁死

**① 哪部分有问题**
方案 §3 写道：「若当天已有任务，enqueue 会因 UNIQUE 冲突返回已存在的任务（幂等）」；预期部分说：「管理员手动触发不会产生重复任务」。

**② 后果**
`background_tasks` 的唯一约束是**全表**的 `UNIQUE (task_type, deduplication_key)`，不含 `processing_state` 的过滤（`migrations/0007_task_type_plugin_discovery.sql:56`）。同时 `enqueue` 的冲突处理是**先 SELECT 后 INSERT，命中就直接返回旧 id**（`src/server/modules/tasks/service.ts:207-211`），不判断旧任务处于什么状态。

两者合起来意味着：只要当天那条 `discovery:github:2026-xx-xx` 任务已经进入 `succeeded` / `failed` / `skipped` **终态**，管理员再点「手动触发」时，拿到的返回值是 `{created:false}` 和一条**已经死了的任务 id**，接口返回 200 看起来成功，实际什么都没执行。也就是说：

- 白天那一轮因为 GitHub 403 限流而 `failed`，运维发现后想手动补跑 → **补不了，当天彻底没办法**，只能改 payload 或动库。
- 发现的结果有错（比如误收了 `tests/e2e` 夹具，`discovery.ts:139` 那段注释就记载过真实事故），人工清理后要重跑 → 同样补不了。
- 更糟的是它会**静默成功**：运维看到 200 就去干别的了，问题留到第二天才被发现。

这是运维场景里最坏的一类故障：**应急通道在需要它的时候恰好是锁死的，而且不报错**。方案把「重复入队被抑制」错误地当成了「人工可以重跑」，二者在终态任务上正好相反。

**③ 建议**
- 手动触发**不能复用** `daily`/`discovery:{source}:{date}` 这个自然日键。改为在键里带唯一性，例如 `discovery:{source}:manual:{requestId}`，让管理员每次触发都是新任务；「防止连点」应该靠前端禁用 + 接口级速率限制，而不是靠永久性 dedup 键。
- 或者显式实现「重跑」语义：`enqueue` 返回已存在的任务时，若其状态属于终态，就**重置该行**（`processing_state='queued'`、`attempt_count=0`、`cycle_attempt_count=0`、`execution_token=NULL`、`available_at=now`、清空 `lease_*`），并把这条逻辑写成独立方法（如 `requeue`），不要塞进 `enqueue` 里污染通用语义。
- 无论选哪种，接口返回值**必须**区分 `created / requeued / still_running(附 taskId 与状态)` 三种，让运维一眼看出「这次点击到底有没有干活」。

---

## 二、高：仓库代码里根本没有 `deduplicationKey` 是可变/可配的，方案却改了键格式 —— 迁移期会双排

**① 哪部分有问题**
方案 §2 定义 `discoveryDedupKey(source, date)` → `discovery:github:2026-05-01`，替换现有的 `daily:2026-05-01`（`src/server/modules/catalog/discovery.ts:115-117` 的 `discoveryDailyKey`）。

**② 后果**
`UNIQUE` 是**按键值**去重的，不认「业务上是不是同一天」。键格式一改，旧键 `daily:2026-05-01` 和新键 `discovery:github:2026-05-01` 在库里是**两条互不相干的行**，唯一约束一点也拦不住。如果发版当天 worker 已经用旧键排过一轮，重启后新代码用新键再排一次 → 同一天对 GitHub 打**两轮**请求。

而这一轮的开销不是小数：认证档 `maxRequests: 400`，未认证档虽然压到 `maxRequests: 58`，但未认证的核心接口只有 **60 次/小时**（`discovery.ts:56` 注释、`discovery.ts:73-79`）。而且**未认证档的 `maxRequests: 58` 本身就是贴着 60/h 上限设计的，没有任何余量**——多排一轮必然 403，把两轮都拖成 `failed`。

顺带一个更值得担心的：方案完全没提 `discoveryBudgetFor`（`discovery.ts:54-80`）这个按 token 分档的预算逻辑。如果新代码里只传 `{source, date}` 的 payload（方案 §2 示例就是这样，`payload: { source, date: ymd(now) }`），把现在 `scheduled.ts:~110` 那套 `run_key / query / search_pages / per_page / max_candidates / max_plugins / max_requests` **全部丢掉**，那 handler 读不到预算就会落到 `DISCOVERY_DEFAULTS` 的**认证档默认值**（`discovery.ts:37-43`：400 次请求、12 候选）。在**没配 token 的线上**，这等于把 58 的保护直接拆成 400 —— `scheduled.ts` 那段注释明确记载过这个坑：「曾经这里写成 camelCase，结果 handler 读不到 → 预算被静默忽略（真实踩过：线上第一轮用了 60 个候选，而不是预算里的 20）」。方案正在重演同一个错误，而且更严重。

**③ 建议**
- **键格式变更必须配一次性数据迁移**，在同一个发版窗口内把 `plugin_discovery` 的历史键前缀 `daily:` 改写为 `discovery:<default_source>:`，否则旧键会永久留在库里当垃圾行（还会一直霸占 `succeeded` 状态，见问题一）。
- 发版顺序固定为：**先停 worker** → 跑迁移 → 起新 worker。或者给新键加一个「兼容读旧键」的去重探测（enqueue 前先按新旧两种键各查一次）。
- payload **必须原样保留** `scheduled.ts` 里那 7 个 snake_case 键，`source`/`date` 只能是**追加**字段、不能替换。建议在方案里显式列出 payload 契约表，并要求 `tests/plugin-discovery.test.ts` 的防漂移用例同步覆盖。
- 未认证档 `maxRequests: 58` 建议再降到 40 左右留余量，否则一次「偶发重排」就足够打爆小时配额。

---

## 三、高：`source` 是管理员输入的字符串，直接拼进 dedup key —— 没有白名单就是留了把刀

**① 哪部分有问题**
方案 §3：`body: { source: "github" }`，实现直接用 `discoveryDedupKey(source, ...)` 拼进键。

**② 后果**
`deduplication_key` 只是 `CHECK (trim(deduplication_key) <> '')`（`0007:35`），对内容**没有任何格式约束**，长度也没有上限。管理员（或拿到管理员会话的人）传 `source: "../../etc"`、几万字符的长串、或者每天换一个随机 source，就能：
- 在 `background_tasks` 里**无限增殖**任务行 —— 因为每个新 source 就是一个新键，天然绕过唯一约束。这不是「幂等」，这是**没有任何速率限制的写放大**。而 `claim` 是 `ORDER BY available_at, created_at, id LIMIT 1` **串行取一个**（`service.ts:126-137`），`MAX_CONCURRENT_TASKS = 4`（`constants.ts:19`）。也就是说灌进去几千条 `plugin_discovery` 之后，会**把用户真实的 `review_check` / `comment_check` 任务饿死**——队列没有优先级，没有公平性，先到先跑。这从「插件目录不更新」升级成「用户提交审核全站卡住」。
- 更隐蔽的：`source` 会进 `run_key`，而 `runKey` 会写进 `operation_records` 的审计（`discovery.ts:90-91`、`scheduled.ts` 注释）。无约束的字符串进入审计表，是取证时的噪声源。

**③ 建议**
- `source` 必须**服务端白名单校验**（`github` 等枚举值，非白名单直接 400），不能透传。方案里需要一个常量表，而不是散在 handler 里。
- dedup key 长度加上限（如 `source` ≤ 32 字符、且 `^[a-z0-9_-]+$`），最好在迁移里补一条 `CHECK (length(deduplication_key) <= 128)`，从库层兜住。
- 手动触发接口需要**独立的速率限制**（复用 `0006_rate_limiter.sql` 那套），例如每管理员每分钟 2 次、全局每小时 10 次。目前方案对此**只字未提**。
- 长期看，`plugin_discovery` 应该有**独立的并发额度**，避免和用户任务抢 `MAX_CONCURRENT_TASKS`。`scheduling_group` 字段已经存在（`DISCOVERY_SCHEDULING_GROUP = 'plugin_discovery_schedule'`），但 `claim` 的 SQL 里**根本没用到** `scheduling_group` 做隔离（`service.ts:126-137` 的 WHERE 只有 state 和 available_at）—— 字段白留了。这一点建议在方案里作为已知缺口写明。

---

## 四、中：方案 §2 的示例代码直接抄会编译失败，`schedulingGroup` 缺失

**① 哪部分有问题**
方案 §2 的 `tasks.enqueue({ taskType, deduplicationKey, requestedBy, payload })` —— 只有 4 个字段。

**② 后果**
`enqueue` 的签名要求 `schedulingGroup: string` 和 `availableAt: number` **都是必填**（`service.ts:56-64`），并且运行时有校验：`schedulingGroup` 为空会 `throw err.input('调度组不能为空。')`（`service.ts:195-197`）。照方案示例写，**第一次调用就抛异常**。虽然这是实施期就能发现的低级问题，但方案作为评审基线，示例代码有误导性；更实际的危害是：实施者会「随手补一个」`schedulingGroup` 字符串，从而丢掉 `DISCOVERY_SCHEDULING_GROUP` 这个**已经在用的观测维度**（`scheduled.ts:34`），事后统计「系统调度 vs 用户任务」就对不上了。

**③ 建议**
方案里的示例补全为 `schedulingGroup: DISCOVERY_SCHEDULING_GROUP, availableAt: now`，并明确它必须复用 `scheduled.ts` 的常量而不是新写字面量。

---

## 五、中：从「每天一轮」变成「每源每天一轮」，是**流量倍增**，方案却当成纯优化

**① 哪部分有问题**
方案标题是「幂等去重改造」，预期里写「每个数据源每天只跑一次」，把它描述成**减少**重复调用。

**② 后果**
现状是每天**一轮**（单键 `daily:{date}`），GitHub 单轮认证档 `maxRequests: 400`、未认证 `58`。改成按 source 分键后，请求量 = **每源预算 × 源数量**，是**随源数量线性增长**。方案通篇没有一处说明「新增一个 source 时上游配额怎么算」，也没有总量上限。按未认证 60 次/小时的核心配额，**2 个源就会打爆**。

也就是说这个改造的真实效果是：**在源数量 ≥ 2 时，它让情况变差了**，而「重复调用 GitHub API」这个原始问题如果根本没定位到根因（是手动触发重复？是 worker 重启重排？），那么分键只会把问题放大。从运维角度看，这是**未经容量评估的变更**。

**③ 建议**
- 方案里补一张**容量表**：源数量 × 每源预算 ≤ 上游配额（含余量），并写明「源数量上限 N，超出必须配 token 或降频」。
- 引入**全局每日请求预算**（跨源共享），例如总 `maxRequests` 上限 400/天，各源按此分摊；而不是每个源各拿一份完整预算。
- 保留 `DSH_REVIEW_DISCOVERY_DAILY=0` 这个硬刹车（`scheduled.ts:57-59`），并**新增按源的刹车**（如 `DSH_REVIEW_DISCOVERY_SOURCES=github` 白名单），出事时能单独关掉某个源而不用全站关停。

---

## 六、中：手动触发的可观测性与审计缺失

**① 哪部分有问题**
方案 §3 只描述了入队，没描述：手动触发的任务**怎么区分**于定时任务、怎么查进度、失败怎么告知发起人。

**② 后果**
上线后运维的第一个问题会是「我刚点了手动触发，跑了吗？跑成什么样？」。现有 `progress` 字段会存结构化结果（`service.ts:71-74` 注释：后台页据此展示「这一轮干了什么」），但方案没说明手动触发时 `run_key` 用什么值 —— 而 `runKey` 正是 `operation_records` 里的审计关联键（`discovery.ts:90-91`）。如果手动触发和定时任务共用 `run_key = discovery:github:2026-05-01`，那么审计里**两轮不同来源的执行会长得一模一样**，事后追查「这轮 400 次请求是谁打的」只能靠猜。

**③ 建议**
- `run_key` 区分来源：定时 `discovery:{source}:{date}`，手动 `discovery:{source}:manual:{requestId}`（同时解决了问题一的键冲突）。
- 接口**同步返回** `taskId`，并提供 `GET /api/admin/discovery/tasks/{taskId}` 查状态；不要只返回 200。
- 失败告警接入现有的通知/审计机制（`0008_notifications.sql`、`0042_audit_immutable.sql` 都在），至少让手动触发的**失败**能主动通知发起人，而不是要他第二天自己翻后台页。

---

## 七、低（但发版必踩）：`0007` 是 `dsh:migration-transaction=off` 的非原子迁移

**① 哪部分有问题**
方案 §1「不新建表，直接在 `background_tasks` 上做」—— 但把 dedup key 加上长度 `CHECK`（我上面的建议）或任何列约束变更，都需要**再次重建表**。

**② 后果**
`0007` 的头部注释写得很清楚：这条迁移自己管事务、PRAGMA 在 BEGIN 之外，**中途失败会留下半成品，执行器不会自动回滚**（`0007:13-16`）。重建表还要 `DROP TABLE background_tasks` 并重建两个索引（`0007:78-83`）。如果这次变更新增 `01xx_*.sql` 走同样的重建路线，一旦失败就是**任务队列表残缺** —— 整个 worker 停摆，所有审核任务卡住。而且 `PRAGMA foreign_keys = OFF` 期间如果进程被杀，外键状态可能残留。

**③ 建议**
- 如果要做表约束变更，**先备份**（`sqlite3 .backup` 或文件级快照），并明确写进发版步骤。
- 可选：用**影子表 + 双写切换**替代直接重建，或者干脆**不在库层加 CHECK**，把 `source` 白名单放在应用层（成本更低、风险更小）。运维角度更倾向后者。
- 任何走 `transaction=off` 的迁移，发版流程里必须有「迁移前后各跑一次一致性检查」的步骤（表行数、索引存在性、`PRAGMA foreign_key_check`）。

---

## 八、结语：方案对「重试」的假设与限流现实冲突

方案 §4 说「沿用 RETRY_BACKOFF_MS = [30s, 120s]、MAX_RECOVERABLE_ATTEMPTS = 3」。但 `discovery.ts:170-176` 那段注释自己就写了：**上游限流时重试无用**，「重试（30s/120s）改变不了配额，只会白烧三次尝试」，所以搜索阶段被限流时是**返回 summary 而不是抛错**。

方案没有说清 `source` 维度的失败语义：某个源因限流失败，是记 `failed`（然后当天没法重跑，见问题一）还是记 `skipped`/`succeeded + truncated`？**这个选择直接决定运维当天有没有救火窗口**。建议明确写进方案：限流类失败应记为「已完成但截断」，保留当天人工重跑能力；只有**真正的程序错误**才进 `failed` 终态。

---

**总结（按修复优先级）**：
1. 手动触发的「假幂等」（问题一）—— 应急通道失效，必须先修；
2. payload 契约被破坏导致预算静默放大（问题二）—— 会直接打爆未认证配额，是已知事故的重演；
3. `source` 无白名单 + 无速率限制（问题三）—— 有饿死用户任务的风险；
4. 容量未评估（问题五）、键迁移（问题二）、可观测性（问题六）。

方案的整体思路（复用队列、按源细分）方向没问题，但**把「幂等」当成了「防重」的同义词，混淆了「抑制重复入队」和「允许人工重跑」这两个相反的需求**，这是最需要在设计阶段掰回来的地方。