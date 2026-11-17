# 评审：发现任务幂等去重方案 · 性能工程师视角

> 评审对象：`atb/tasks/m10-idempotent-queue/environment/proposal.md`（46 行）
> 评审人身份：**性能工程师**（只看容量、吞吐、上游配额、队列时序、写放大、索引与运维成本；不评权限模型与合规判定）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**按现方案落地会让「同一天不再重复调用 GitHub API」这个目标彻底落空**，并且把配额消耗放大 N 倍。
> 方案在 §2 把去重粒度从「天」改成「天 × 数据源」，而它要解决的正是「同一天被多次触发」——
> 前者放宽了约束、后者要求收紧约束，两者方向相反。

---

## 摘要（前 5 条最重要）

1. 【严重】**§2 的 dedup key 改窄 = 把「一天一轮」变成「一天 N 轮」，恰好放大它想消灭的重复调用。**
   依据：`proposal.md:12-18`（`discovery:${source}:${date}` 取代 `daily:${date}`）；现状 `src/server/modules/catalog/discovery.ts:121-124`（`discoveryDailyKey` 返回 `daily:YYYY-MM-DD`）。
   数值后果：一轮的**自带预算**是 `maxRequests: 400`（认证）/ `58`（未认证），见 `discovery.ts:37-44` 与 `:55-81`；
   单轮典型开销 `≈ 搜索 1-2 + 每仓库 1 次 tree + N 个 package.json + 每收录 3 次`（`:33-35` 注释）。
   未认证下 GitHub 核心配额是 **60 次/小时**（`:49-51`、`deploy/discovery-drip.sh:2-3`），今天之所以只排一轮，
   正是因为这一轮就能吃掉约 26~58 次（`discovery-drip.sh:21-22` 实测「单轮 ~26 次核心请求」）。
   **按 source 拆成 N 份 = N × 26 次**，未认证环境当天就会 403 收尾（`discovery.ts:170-187` 的限流分支）。
   建议：**保持「一天一个预算桶」**（单一 daily key），把 source/query 作为**轮换维度**放进 payload，
   由 `deploy/discovery-drip.sh:12-25` 那套「按时间槽轮换搜索词」承担多源覆盖，而不是并行开 N 个任务。

2. 【严重】**§3「手动触发用当天的 dedup key」= 管理员永远无法手动再跑一轮，而且不会报错。**
   依据：`proposal.md:38`；`src/server/modules/tasks/service.ts:190-226` —— `enqueue` 命中 `(task_type, deduplication_key)`
   时返回已存在的 taskId 且 **`created: false`、不重置进度、不改状态**（`:205-208`）。
   后果：当天自动那一轮跑完之后（哪怕它已是 `succeeded`/`failed` 终态），管理员点「立即发现」拿到的是一个
   **早就不动的旧任务 ID**，接口照样 202（`proposal.md:38` 自称「幂等」），前端看起来一切正常。
   这在本仓是**已被刻意区分过的语义**：定时用 `daily:`（`src/worker/scheduled.ts:79,103-105`），手动用 `manual:${requestKey}`（`src/app/api/admin/discovery/route.ts:108-113`）。
   建议：手动触发继续用独立 key（`manual:<requestKey>` 或 `manual:<source>:<ts>`），§3 的例子改成与现有
   `POST /api/admin/discovery` 同形（`requestKey` 由前端生成，重放同一 requestKey 才幂等）。

3. 【高】**改 key 格式没有迁移旧行，历史与新版会同时存在；而重试/重跑会各自烧一份配额。**
   依据：`proposal.md:43-45` 只说了「每源每天一次」；`migrations/0007_task_type_plugin_discovery.sql:47` 的 `UNIQUE (task_type, deduplication_key)` 是**按 task_type 全局**的，
   旧 `daily:2026-11-15` 行与新 `discovery:github:2026-11-15` 行**不冲突**，于是同一天两批任务都会跑。
   更关键的是任务进 `failed` 后**不会被再去重**：重试路径（`service.ts:59-73` 的 finish、`constants.ts:59-64` 的 `nextAvailableAt`）
   允许同一行在 30s/120s 后重跑；发现任务的重试语义是「网络抖动/上游限流/DB busy → error → 重试」（`src/worker/handlers.ts:282-285`），
   而**上游限流重试是白烧配额**（`discovery.ts:171-173` 已写明「重试（30s/120s）改变不了配额，只会白烧三次尝试」）。
   建议：把「限流」在 handler 层映射成终态 skipped（见 P-06），并在方案里写明旧 key 的处置。

4. 【高】**没有「同一时刻只让一个发现任务在跑」的闸门，手动 + 定时会叠加成两份消耗。**
   依据：`proposal.md:28`、`:43-45`；`src/worker/main.ts:190-205` —— 每轮 `for (i < maxConcurrent) claim()`，
   `MAX_CONCURRENT_TASKS = 4`（`constants.ts:16`）。两个 key 不同 → 两行 → 可能在同一批里先后被领走；
   而 `scheduling_group` **没有串行语义**（只在 `0007:80` 建了索引，`claimWithin` 的排序见 `service.ts:122-131`，与组无关）。
   后果：worker 是顺序执行（`main.ts:199-204`），但两轮串起来仍消耗 2×26 次核心请求；
   在 300s 截止 + 120s 租约下，第二轮还容易反复被 `recoverExpired` 回收重排（`main.ts:173`）。
   建议：加单实例互斥（固定 key `discovery:lock`，或 enqueue 前查「是否已有 queued/running 的 plugin_discovery」），
   并在 `GET /api/admin/discovery` 里显式暴露「当前有一轮在跑」。

5. 【中】**§4「沿用 RETRY_BACKOFF_MS / MAX_RECOVERABLE_ATTEMPTS」与发现任务的真实耗时分布不匹配。**
   依据：`proposal.md:40-41`；`constants.ts:19-45`（`LEASE_DURATION_MS=120s`、`TASK_DEADLINE_MS=300s`、`RETRY_BACKOFF_MS=[30s,120s]`）；
   `src/server/adapters/catalog/github-discovery.ts:86-87`（**单次上游超时 20s**）、`:151-155`（`retries: 2` → 单请求最多 3 次尝试）。
   后果：一轮要串行访问十几个仓库（`discovery.ts:41` `maxCandidates: 12`），最坏 `12 × 3 × 20s` 远超 300s 截止；
   超时后 `recoverExpired` 重新排入（`main.ts:173`），**下一轮从头再烧一遍**——`requestsUsed()` 是**适配器实例级**计数
   （`github-discovery.ts:160,304`），每次重跑从 0 开始，预算不跨次累计。建议见 P-05。

---

## 完整问题清单（按严重度）

### P-01 【严重】去重粒度细化 = 配额消耗 × N，与方案目标相反
- **现场**：`proposal.md:12-18`「把去重粒度从「天」细化到「数据源」」「这样每个数据源每天一个任务，互不干扰」。
- **现状代码**：`discovery.ts:121-124` `discoveryDailyKey(now) → 'daily:' + YYYY-MM-DD`；
  一轮预算常量在 `discovery.ts:37-44`（`maxCandidates: 12 / maxPlugins: 30 / maxRequests: 400`），
  未认证分档在 `:71-80`（`maxCandidates: 10 / maxPlugins: 12 / maxRequests: 58`）。
- **后果（可算的）**：
  - 未认证 60 次/小时、单轮实测约 26 次核心请求（`deploy/discovery-drip.sh:21-22`）→ 两个 source 就 52 次，加一次手动即 403；
  - 认证环境 5000 次/小时虽宽，但**搜索接口是独立限额**（`discovery.ts:72-73`：匿名 10 次/分），多源并发先把搜索端打满；
  - 这与 `discovery.ts:14-16` 写明的纪律「它有自己的上游预算，超额就如实截断」相冲突：方案把**预算**留在单轮，却把**轮数**放大 N 倍。
- **建议**：一天一个任务、一个预算桶；source 只作轮换调度维度：
  ```ts
  deduplicationKey: discoveryDailyKey(now),   // 复用现状，不动 UNIQUE 语义
  payload: { run_key, query: rotateQuery(now), ...budget },
  ```
  轮换直接复用 `deploy/discovery-drip.sh:20-25` 的「按时间槽取第 idx 个搜索词」（已有线上实践）。
  若确实要多源并行，必须同时把 `maxRequests` **按源数均分**（`budget.maxRequests / N`），否则等价于把预算乘以 N。

### P-02 【严重】手动触发复用当日 key = 静默无效
- **现场**：`proposal.md:33-38`。
- **依据**：`service.ts:205-208`（命中即 `{ taskId: existing.id, created: false }`，**不推进任何状态**）。
- **后果**：管理员在自动轮跑完后点「立即发现」→ 拿到旧 taskId，HTTP 仍 202；后台页 `src/app/api/admin/discovery/route.ts:22-48`
  按 `created_at DESC` 展示，**看不到新行**，以为「点了没反应」而反复点。若旧行已是 `failed`，它**永远不会被重新领取**，
  管理员等半小时也不会开始任何一轮——静默无操作比报错更消耗运维时间。
- **建议**：手动走独立 key，直接沿用 `route.ts:108-113` 的既有实现（`manual:${requestKey}`）；`source` 只决定 query，不参与 key。

### P-03 【高】老 key 未处理：同一天新旧两批任务并存
- **现场**：`proposal.md` 全文没有迁移/兼容说明。
- **依据**：`0007_task_type_plugin_discovery.sql:47`；变更是**纯应用层**（方案自称「不新建表」），库里没有任何约束能统一两种格式。
- **后果**：发布当天 `daily:2026-11-15`（已存在）与 `discovery:github:2026-11-15`（新）各跑一次，当天多烧一轮；
  之后 `GET /api/admin/discovery`（`route.ts:29-37` 取最近 10 条）里新旧混排，「最近几轮」可读性下降。
- **建议**：方案里写明「本次为应用层变更，旧 key 行保留不动，发布日可能出现新旧各一轮」，
  并把发布安排在当天额度已用尽之后（如北京时间 23:00 后）；或在 `enqueue` 处对两种键做一次探测（多一次 `SELECT`，可忽略）。

### P-04 【高】无单实例闸门：手动 + 定时会被同一批 claim 连跑
- **现场**：`proposal.md:28`、`:43-45`。
- **依据**：`src/worker/main.ts:190-205`（一轮最多 claim 4 个）、`constants.ts:16`；
  `scheduling_group` 只在 `0007:80` 建了索引，`service.ts:122-131` 的 claim 排序是 `available_at, created_at, id`，**没有按组互斥**。
- **后果**：两个 key → 两行 queued → 同一批内被领走；worker 顺序执行（`main.ts:199-204`）意味着串行，但**消耗仍是 2×**；
  且在 300s 截止/120s 租约下第二轮常被 `recoverExpired` 重排（`main.ts:173`），形成「重排—再跑—再超时」抖动。
- **建议**：加互斥键（`discovery:lock`）或 enqueue 前查「是否已有 queued/running 的 `plugin_discovery`」。

### P-05 【中】§4 的重试参数与发现任务耗时分布不匹配
- **现场**：`proposal.md:40-41`。
- **依据**：`constants.ts:19/22/25/45`（租约 120s / 续期 30s / 恢复扫描 10s / 截止 300s）；
  `github-discovery.ts:87`（`DEFAULT_TIMEOUT_MS = 20_000`）、`:151-155`（`retries: 2`）。
- **后果**：单仓库最坏 3×20s=60s，`maxCandidates` 默认 12 → 最坏远超 300s；超时重排后**从头再跑**（`requestsUsed` 实例级，`:160`），
  把已花的配额白扔一次再花一次——正是 `discovery.ts:171-173` 想避免的事。
- **建议**：① 发现任务单独放宽截止（如 900s），或按剩余时间动态收紧 `maxCandidates`；
  ② 把「本轮已用请求数」写进 payload/progress，重跑时**从剩余额度继续**而不是从 0 开始；
  ③ §4 的表述改成「重试只修可能自愈的故障」——限流不是（见 P-06）。

### P-06 【中】限流走重试路径：3 次尝试 × 2 轮退避都在白等
- **现场**：`proposal.md:40-41` 把重试作为唯一失败恢复手段。
- **依据**：`discovery.ts:143-145,169-187`（限流时**不抛错**，返回 `truncated: true` 的干净收尾）；
  `handlers.ts:353-359`（其它错误 → `outcome: 'error'` → 队列按 30s/120s 重试）；
  `github-discovery.ts:193-203,240-255`（403/429 与「仓库被屏蔽的 403」已分开判定）。
- **后果**：某轮若因非搜索阶段的限流抛错，队列会 30s、120s 各重试一次，两次都必然失败，共浪费 2 次完整跑动。
- **建议**：handler 里读 `adapter.wasRateLimited()`（已在 port 暴露，`github-discovery.ts:74-81`），为 true 时返回
  `terminal('discovery_rate_limited', …)`（skipped），与 `provider_not_configured` 同口径（`handlers.ts:356-358`）。

### P-07 【中】「按 source 拆分」与既有「按 query 轮换」是两套互不兼容的多源策略
- **现场**：`proposal.md:12-14` 引入 `source`；仓库里**不存在**这个维度。
- **依据**：`discovery.ts:37-44` 只有单一 `query: 'topic:dsh-plugin'`，多源覆盖靠轮换 query：
  `deploy/discovery-drip.sh:12-18` 列了 5 个搜索词、`:20-25` 按 40 分钟槽轮换；`handlers.ts:309` 从 payload 读 `query`。
- **后果**：两套机制并存时会出现「同一个 GitHub 上游被拆成 source=github 与 source=github-dsh 两个 key」这种自欺的分法；
  **配额从来是按上游（GitHub API）计的，不是按我们的命名计的**。
- **建议**：若引入 `source`，先定义为「真实外部数据提供方」（GitHub / Gitee …），并与 `DISCOVERY_DEFAULTS.query` 的轮换表合并；
  不要出现两个 source 指向同一上游。

### P-08 【低】索引无需改，但 key 基数会上升
- **现场**：`proposal.md:9-10`「不新建表，直接在 background_tasks 上做」。
- **依据**：唯一约束已覆盖（`0007:47`）；查询侧 `idx_tasks_claimable(processing_state, available_at, scheduling_group, id)`（`:80`）
  由 `service.ts:122-131` 使用，**与 dedup key 无关**，key 变长/变多**不影响索引选择**。
- **后果**：可忽略；实际影响只有历史行更多（P-03）与后台列表混排。
- **建议**：无需加索引；若要可读性，可在查询里加 `AND deduplication_key LIKE 'discovery:%'`。

### P-09 【低】无「今天跑了几轮、烧了多少请求」的汇总度量
- **现场**：`proposal.md:43-46`「预期」三条都是语义描述，没有度量。
- **依据**：现有可观测手段是任务 `progress`（`handlers.ts:352` 写 `summary`，含 `requestsUsed`）与审计 `plugin.discovery.run`（`handlers.ts:319-345`）。
  一天变成 N 轮后，**没有任何地方汇总当日总请求数**；`discovery.ts:320-332` 每轮打一行日志，没有当日累计。
- **后果**：配额事故（403 收尾）只能事后逐轮相加日志。
- **建议**：`GET /api/admin/discovery` 返回体加「今日各 source 轮数 + `SUM(json_extract(progress,'$.requestsUsed'))`」，
  一次聚合、量级为当日行数，代价可忽略。

---

## 结论与两条落地建议

**现方案在性能/配额维度不可放行**：它把「一天一次」改成「一天 N 次」，而在未认证环境下配额是一堵**硬墙**
（`discovery.ts:49-51`、`deploy/discovery-drip.sh:2-3`），不是「慢一点」，而是「当天剩余时间全部 403」。

**建议 A（最小改造、真正解决问题）**——不改 dedup key 的天粒度，只补三件事：
1. 手动触发用独立 key（`manual:<requestKey>`，已有实现 `src/app/api/admin/discovery/route.ts:108-113`）；
2. 加单实例闸门（避免手动 + 定时同日并发/连跑）；
3. 多源覆盖靠 payload 里的 `query` 轮换（复用 `deploy/discovery-drip.sh:20-25`），不靠多开任务。
   这样「每数据源每天只跑一次」以**零配额增量**实现。

**建议 B（若产品坚持要 source 维度）**——必须同时满足：
- `maxRequests` 按 `sources.length` **均分**（否则预算被乘 N）；
- 全局每日请求上限落库（跨轮累计），超限即 skipped；
- 单实例闸门 + 限流映射成终态（P-04、P-06）。
  缺任何一条，这个方案就是把「重复调用 GitHub」的小问题，换成「配额耗尽、发现停摆一整天」的大问题。

---

> 评审人声明：本报告只覆盖性能/容量/上游配额/队列时序视角。`source` 的白名单校验、权限模型与审计合规由其他视角负责，
> 本文不代替它们的结论。方案「背景」一节（`proposal.md:4`）与仓库一致：`discoveryDailyKey` 确为 `daily:YYYY-MM-DD`
> （`src/server/modules/catalog/discovery.ts:121-124`），前提成立。
