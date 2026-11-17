# n27-webhook-delivery SRE（运维）评审

> t21 · n46-sre · 2026-11-17

---

N27（插件更新 Webhook 推送）SRE 评审完成：结论「有阻塞项（4 P0 + 4 P1），不建议按现行方案上线」。报告：_work/评审-N27-Webhook推送-SRE.md

P0-1 「插件更新时异步 POST」没有落点。写进 web 请求路径：裸 fetch 无超时（http.ts:15-22 的 8s 只属于那个工具）、无并发上限，订阅者慢/挂就把"更新插件"请求拖到 TCP 超时。写 fire-and-forget：compose restart:unless-stopped（compose.yaml:68,93）+ 镜像更新必重启，in-flight 推送静默丢失且无痕迹。项目所有外部副作用都走队列（handlers.ts:369-377、main.ts:190-205 串行+租约），方案却绕开手写。
P0-2 task_type 是 CHECK 闭集（0001:347-348、0007:25-26、field-rules.ts:88-95，tests/migrate-transaction.test.ts:99-115 断言），新类型要重建表且 migrate.ts:97-115 明说该类迁移失败留半成品；compose.yaml:27-42 的 migrate 前置容器与 worker.service:8 的 After= 都挡不住旧进程持连接。若不新增类型而手写重试：大概率连"重试也要带租约"都没有，两个 worker 会同时重试同一事件。建议：新增 webhook_delivery 类型 + 按订阅者去重键 webhook:<subId>:<eventId>，用 UNIQUE(task_type, dedup_key) 一次解决去重与重试。
P0-3 无死信/无订阅降级/无熔断：3 次失败后事件永久丢失；失败预算按整条任务累计（service.ts:328-331），坏订阅持续产生 failed；worker 只有 4 个并发名额且串行执行（constants.ts:16、main.ts:190-205）→ 会把 review_publish 这类用户可见任务挤在后面；一次对方 502 十分钟就留下成百上千条 failed，而全仓无任务保留策略。
P0-4 secret 存明文且根本没用：方案存了 secret 却不签名，接收方无法验证来源；明文进库后会随 backup.mjs:62-70 进入 14 份历史备份（compose.yaml:62）；无 secret_secondary/rotated_at 轮换、无事件 id+timestamp 防重放。

P1-5 可观测性为零：diagnose.mjs:241 只看 problems、无 webhook 项；health.ts:1-9 明确不做业务统计 → 推送全线失败时站点一切"正常"，而第三方不会来报障。建议结构化日志 {subscription_id,event_id,attempt,status,ms} + operation_records + diagnose 判据。
P1-6 重试风暴：无退避、无单订阅节流；沿用 [30s,120s]（constants.ts:32-39）对正在重启的对端几乎必再失败还占 worker 名额。建议 [1m,5m,30m,6h] + 每域令牌桶 + 24h 后标记过期失败。
P1-7 网络面：现状出网目标是写死的几个站（github-probe.ts:68、github-discovery.ts:84、verify.ts:26、parse-url.ts:5-8），compose.yaml:135-137 无 egress 限制；支持用户填 URL 后变成"从内网向任意地址 POST"，且 webhook 出网不经 nginx → 入口日志一条都没有，排障无线索。建议 https + 管理员白名单 + 出网目标巡检。
P1-8 关不掉/回滚不了/迁移编号与备份未定：现有刹车都是 env（scheduled.ts:16-17,45-53、.env.example:63），webhook 无开关；回滚后库中留无人消费类型（main.ts:91-104 判 skipped 不告警）；仓库到 0041、migrate.ts:51-55 按文件名排序且无重复编号检测；backup.mjs:78-88 行数清单无新表项。

P2-9 表结构：缺 UNIQUE(user_id,url)、url 的 https CHECK（先例 featured/index.ts:217）、disabled_at/failure_count/last_error、deleted_at、user_id 索引。

另附 8 项上线验收清单（不在请求路径/重启不丢/坏订阅不拖垮队列/自动降级/签名可验证/失败可见/关得掉/备份可对账）。