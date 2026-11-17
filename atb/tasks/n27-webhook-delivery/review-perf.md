# 评审：插件更新 Webhook 推送 · 性能工程师视角

> 评审对象：`atb/tasks/n27-webhook-delivery/environment/proposal.md`（37 行）
> 评审人身份：**性能工程师**（只看扇出成本、阻塞与超时、重试放大、索引与磁盘、worker 并发位、远端压力；不评签名与合规实现）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**方案里三行 `fetch` 会把「插件更新」从一个 O(1) 的写操作，放大成「订阅者数量 × 3 次重试 × 每个超时」的串行网络风暴**，
> 而它既没有超时、也没有上限、也没有走仓库既有的队列与退避。

---

## 摘要（前 5 条最重要）

1. 【严重】**§2 的 await fetch(...) 写在「插件更新时」——同步扇出，且没有超时。**
   依据：`proposal.md:22-30`；本仓所有出网调用都带 AbortController + 显式超时：`src/server/platform/http.ts:15,21-22`（默认 8000ms）、
   `src/server/adapters/catalog/github-discovery.ts:86-87`（后台任务 20s，注释写明「比用户路径的 15 秒更长」）；
   `src/server/platform/db.ts:6-8` 明确纪律「业务事务保持短小，不在事务内做网络请求」。
   后果：一个更新事件 × N 个订阅者 = N 次串行 fetch。无超时意味着**一个挂住的接收方可以无限期挂住调用方**；
   若在收录/更新插件的事务里，SQLite 写锁被持有到网络返回为止——全站写入排队（单写者）。即使在事务外，放在 web 请求路径上同样致命：
   node fetch 不设超时会等到 TCP/服务器超时（可数分钟）。
   建议：① 路由/服务层只写一条待投递记录（同事务、O(1)）；② 真正 POST 由 worker 用已有队列做（见摘要 3）；
   ③ 任何出网调用都必须带 AbortController + 超时（5-10s 足够，webhook 接收方不该更慢）。

2. 【严重】**§3「失败重试 3 次」没有退避，等于把失败转换成 3 倍峰值流量。**
   依据：`proposal.md:32-33`（只有一句话）；本仓已有的退避语义：`src/server/modules/tasks/constants.ts:39`（RETRY_BACKOFF_MS = [30s, 120s]）、
   `:59-64`（nextAvailableAt：第 1 次失败等 30s，第 2 次等 120s，第 3 次直接终态）；
   先例写在代码注释里：`src/server/adapters/catalog/github-discovery.ts:171-173`「重试（30s/120s）改变不了配额，只会白烧三次尝试」。
   后果：接收方处于故障窗口（重启、限流）时立即重试 3 次 = 3 倍请求打在同一时刻；N 个订阅者 × 3 = 3N 次外部请求，
   全部压在 worker 的 4 个并发位里（`src/worker/main.ts:190-205`、`constants.ts:16`）。
   建议：复用 30s/120s 形状或指数退避 + 抖动；并区分可重试与不可重试——对方 4xx 不重试，只有 5xx/超时/连接失败才重试
   （与 `github-discovery.ts:169-176` 的「确定性失败不重试」同一条纪律）。

3. 【严重】没有用仓库既有的任务队列（background_tasks），手写重试绕开了租约/退避/恢复扫描。
   依据：`proposal.md:22-33`；`src/server/modules/tasks/service.ts:116-180`（原子 claim + 租约）、`:278-395`（finish 写回）、`:402-440`（recoverExpired）；
   `src/packages/common/field-rules.ts:87-96`（TaskTypes 是封闭集合，注释「不能由配置注入新类型」）；
   `migrations/0007_task_type_plugin_discovery.sql:1-19`（新增 task_type 需要重建表，PRAGMA foreign_keys=OFF）。
   后果：不用队列就得自己解决「重试状态放哪、进程重启怎么办、并发怎么限」——仓库里都已经解过一遍；
   用队列则要新增 task_type 并重建 background_tasks（全表 INSERT SELECT + DROP + RENAME，`0007:62-81`），这是发布期的停写窗口，方案一字未提。
   建议：投递拆成「事件表（追加）+ 每订阅者一行投递任务」；任务类型用一个通用 webhook_deliver（一次重建长期够用）；
   **一个任务只投递给一个订阅者**，失败重试、并发限流、自动禁用都落在一行任务上，成本可算。

4. 【高】webhook_subscriptions 没有任何索引与约束——投递路径上每次事件都要全表扫。
   依据：`proposal.md:11-19`（只有 PK；user_id 无索引、url 无唯一约束、没有 enabled 状态列）；
   本仓同类表常态写法：`migrations/0040_user_roles.sql:64-65`（role_applications 带 idx_..._user 与 idx_..._queue）、`:69-70`（部分唯一索引防重复待审）。
   后果：每个更新事件一次全表扫；而「插件更新」是目录最高频事件之一（plugins.info_updated_at 每次同步都会动，`0005_plugin_directory_info.sql:48-52`），全表扫会被乘以事件频率。
   没有 enabled 列也无法在失败后停用订阅，只能靠每次超时去发现（见摘要 5）。
   建议：CREATE INDEX idx_webhook_subs_user ON webhook_subscriptions(user_id, created_at, id)；
   加 enabled / failure_count / last_success_at / disabled_at 四列；再加部分唯一索引 uq_webhook_subs_url ON (user_id, url) WHERE enabled = 1（同 0040:69-70 手法）。

5. 【高】没有「长期失败自动停用」→ 失效订阅永久占用 worker 的并发位与截止预算。
   依据：`proposal.md` 全文无停用/隔离机制；`src/worker/main.ts:190-205`（每轮最多 4 个、串行）、
   `src/server/modules/tasks/constants.ts:45`（TASK_DEADLINE_MS = 300s）、`service.ts:402-440`（超时回收后仍会重排）。
   后果：积累一批失效 URL 后，每次事件都要付出「超时 × 重试」的时间；worker 的 4 个并发位被永远失败的投递占满，
   评测发布、评论发布跟着排队（这是本仓唯一的执行通道）。
   建议：failure_count 连续 N 次（如 10）失败 → enabled = 0 并可在后台一键恢复；投递并发单独限流（如最多 2 个），不与内容发布抢同一个池。

---

## 完整问题清单（按严重度）

### P-01 【严重】同步扇出 + 无超时：一次更新 = N 次串行外网调用
- 现场：`proposal.md:22-30`。
- 依据：`src/server/platform/db.ts:6-8`、`src/server/platform/http.ts:15,21-22`、`github-discovery.ts:86-87`。
- 后果：N=100 个订阅者、每个 8s 超时 → 单次更新最坏 800 秒外部等待；若在 web 请求路径上，网关早已断开（客户端 504），
  事件却可能推送了一半——没有幂等键就无法安全重放。
- 建议：见摘要 1/3；给每个事件一个 event_id 放进 payload，接收方可去重、我们可安全重放。

### P-02 【严重】重试无退避、无分类，3 倍放大失败流量
- 现场：`proposal.md:32-33`。
- 依据：`src/server/modules/tasks/constants.ts:39,59-64`；`github-discovery.ts:169-176`。
- 后果：无抖动时多个订阅者同时重试，形成对齐尖峰（同一公司网关后挂 50 个订阅 = 50×3 同刻并发）。
- 建议：delay = base * 2^attempt + random(0, base)；4xx 不重试；5xx/超时/连接错误才重试。

### P-03 【严重】绕开既有队列；若走队列则要重建 background_tasks，方案未提成本
- 现场：`proposal.md:21-33`。
- 依据：`src/server/modules/tasks/service.ts:116-180,278-395,402-440`；`src/packages/common/field-rules.ts:87-96`；`migrations/0007_task_type_plugin_discovery.sql:1-19,62-81`。
- 后果：TaskTypes 是编译期封闭集合，新增类型要同时改 field-rules.ts:88-95 与迁移 CHECK（0007:26），两处不同步会直接 500。
- 建议：通用 webhook_deliver 任务类型 + 每订阅者一行；deduplication_key 用 webhook:<eventId>:<subscriptionId>（0007:47 的唯一约束天然保证同一事件同一订阅只投一次）。

### P-04 【高】表缺索引/约束/状态列
- 现场：`proposal.md:11-19`。
- 依据：`migrations/0040_user_roles.sql:64-65,69-70`；`migrations/0005_plugin_directory_info.sql:48-52`。
- 后果：url 无唯一约束时，同一用户可对同一 URL 配 100 条订阅 → 一次事件推 100 次到同一目标。
- 建议：见摘要 4 的索引与列。

### P-05 【高】无「停用长期失败订阅」→ 永久占用 worker 并发位
- 现场：`proposal.md` 全文。
- 依据：`src/worker/main.ts:190-205`、`src/server/modules/tasks/constants.ts:16,45`、`service.ts:402-440`。
- 后果：见摘要 5。
- 建议：failure_count + enabled + 自动停用 + 投递并发单独限流。

### P-06 【中】投递记录无保留期：只增不减，且按「事件 × 订阅数」乘数增长
- 现场：`proposal.md` 无 TTL/清理设计（§1 只建了订阅表，事件与投递记录没提）。
- 依据：本仓对只增表的态度见 `migrations/0042_audit_immutable.sql:17-27`（审计追加不可删），但投递记录不是审计，没有理由永久保留；
  `scripts/backup.mjs:78-94` 会对关键表做行数对账与整库备份。
- 后果：一年后备份体积与 integrity_check 时长同步上涨。
- 建议：投递记录保留 N 天（如 14）后批量删除；删除走一次性批处理（worker 每轮开头），**不要 setInterval**——
  与 `src/server/platform/rate-limit.ts:182-184` 的纪律一致（原文：「不开启库的定时清理：它会 setInterval，而我们要进程能被干净关掉」）。

### P-07 【中】payload 体积与内容边界未定义
- 现场：`proposal.md:28`（body: JSON.stringify(event)）。
- 依据：`src/server/platform/http.ts:4-6`（不让第三方返回体撑爆内存的既有约束）；`0040_user_roles.sql:55,57`（文本列一律加 length CHECK）。
- 后果：事件体若带 README/描述全文可达数百 KB，N 个订阅者 × 3 次重试 = 同一份大 JSON 反复序列化 N×3 次；
  若事件体存进任务 payload，会把最热的表撑大（claimWithin 每次都取 payload，`service.ts:122-131`）。
- 建议：payload 只放标识与摘要（event_id / plugin_id / version / occurred_at / url），正文让接收方回查；单条 payload 上限 4KB。

### P-08 【低】投递结果无汇总可观测
- 现场：`proposal.md:35-37`。
- 依据：本仓可观测手段是任务 progress 与 operation_records（`0001_baseline.sql:411-431`）。
- 后果：订阅数、失败率、平均投递耗时都无从查询。
- 建议：webhook_subscriptions 上加 last_success_at / last_failure_at / failure_count 三列，后台页直接读（一次带索引的查询），不需要额外统计表。

---

## 结论与建议

**方案性能上不可按原样落地**：三行 fetch 掩盖了三件必须显式设计的事——在哪里执行、失败怎么退避、失效订阅怎么隔离。
而这三件本仓都已经有现成答案（background_tasks 的租约/退避/恢复、http.ts 的超时范式、0040 的索引与状态列范式）。

**最低改造（保持方案结构不变）**：
1. 事件 → 追加一行事件记录；每个**启用**的订阅 fan-out 出一行投递任务（dedup = webhook:<eventId>:<subId>）。
2. 投递由 worker 执行，出网必带 AbortController + 5-10s 超时；4xx 不重试，5xx/超时才退避重试（30s/120s）。
3. webhook_subscriptions 补 enabled / failure_count / last_success_at / disabled_at + (user_id, created_at, id) 索引 + (user_id,url) WHERE enabled=1 部分唯一索引。
4. 连续失败 N 次自动停用；投递并发单独限流，不占满 worker 的 4 个并发位。
5. 投递记录保留 14 天后一次性批处理删除（不 setInterval）；payload 只放标识与摘要（≤4KB）。

> 评审人声明：本报告只覆盖性能/容量/阻塞/重试放大/运维成本视角。SSRF（url 由用户填、服务端代替访问）与 HMAC 签名属于安全视角，
> 本文不代替它们的结论；但请注意 P-01 的结论对安全同样成立：**同步无超时的出网 fan-out 是两类风险共同的放大器**。

> 前提核实：§1 的 webhook_subscriptions 是声明新建的表（合理）；§2 的 fetch 与 §3 的「重试 3 次」在仓库中没有对应实现；
> §2/§3 也没有引用 background_tasks（`0001_baseline.sql` 与 `0007` 中真实存在），属于「该用而没用」。
