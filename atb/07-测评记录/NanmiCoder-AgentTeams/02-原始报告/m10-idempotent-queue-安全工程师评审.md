# m10-idempotent-queue 安全工程师评审

> 任务 t7 · 成员 e3-investigator · 状态 completed
> 日期：2026-11-17

---

# m10-idempotent-queue 安全工程师评审

评审对象：atb/tasks/m10-idempotent-queue/environment/proposal.md（46 行）
基线核实：background_tasks 真实存在于 migrations/0001_baseline.sql:345-384，0007 已放开 task_type。

## 最重要 5 条

### S1【阻塞】方案对“幂等”的描述与真实行为不符，会掩盖一个真实的吞任务缺陷
proposal:38 写「enqueue 会因 UNIQUE 冲突返回已存在的任务（幂等）」。
**代码事实**：tasks/service.ts:205-208 是**先 SELECT 再 INSERT**（不是靠 UNIQUE 冲突），
命中就 return { taskId: existing.id, created: false } —— **不抛错、不重置状态、不重跑**。
后果：提案的新键是 discovery:<source>:<date>（proposal:16-18），**不含 requestKey**。
于是管理员当天第一次点「立即发现」后，该 source 当天再点任何次数都**永远只是拿回那个已结束的任务**
（可能已是 succeeded/failed），按钮看起来“成功”但什么都没跑。
现有实现正是为此才用 manual:<requestKey>（src/app/api/admin/discovery/route.ts:109）——
proposal:38 的说法会让实现者以为“冲突=幂等正确”，从而**删掉 requestKey 这一段**，把现有可用行为降级成静默失效。
建议：显式区分两种键。定时轮用 discovery:<source>:<date>；手动轮必须继续带 manual:<requestKey>
（或 discovery:<source>:<date>:manual:<requestKey>），并在文档写明“enqueue 返回 created=false 时必须查状态并告知用户，不能报成功”。

### S2【高】新路由 POST /api/admin/discovery/trigger 与既有路由重复，且丢掉幂等凭据
proposal:34 新建 POST /api/admin/discovery/trigger。
**代码事实**：已存在 POST /api/admin/discovery（src/app/api/admin/discovery/route.ts:86-132），
已实现 assertCanAdmin（:90）、requestKey 必填校验（:93-95）、maxPlugins 上下界（:101-106）、幂等键（:109）。
提案等于**再开一个入口**，而新入口 body 只有 { source }（proposal:35），**没有 requestKey**
→ 无法区分“同一意图的重试”与“两次不同意图”，也会形成第二条需独立审计/限流的写路径。
建议：不新建路由，改扩展现有 POST 接受可选 source；若确要新路由，必须同样要求 requestKey、
复用同一 service 调用，并在 admin 审计里留 actorUserId。

### S3【高】source 无枚举、无白名单，直接进 dedup key 与 payload
proposal:15-18 的 discoveryDedupKey(source: string, date: string) 接收任意字符串。
deduplication_key 列只有 trim(...) <> ''（migrations/0001_baseline.sql:349），无长度/字符集限制。
后果：(a) 管理员（或任何持有该 admin 通道的人）可传入超长任意文本，每换一个 source 就**多一条任务**，
“每源每天一个”的承诺失效；(b) 若 source 参与 URL/查询拼接（proposal:35 语义指向 GitHub 数据源），
则是未校验的注入面；(c) 键里用 ':' 作分隔，若 source 自带 ':' 会与 discovery:<source>:<date> 产生**键碰撞**
（如 source='a:2026-11-17' 与另一组同日键重合）。
建议：source 定义为受控枚举（如 'github' | 'github_topic'），落在 packages/common/field-rules.ts，
服务层与路由层双重校验；键分隔符改为不会被数据包含的字符，或对 source 做转义。

### S4【中】新键改变既有去重语义，历史任务不再被识别 → 可能同一天跑两轮
proposal:4 现状键是 daily:<date>（discovery.ts:122-124），proposal:17 改为 discovery:<source>:<date>。
**同一 task_type=plugin_discovery 下**，升级当天会有两条不同的键：已存在的 daily:2026-11-17
与新的 discovery:github:2026-11-17 —— UNIQUE(task_type, deduplication_key)（0001:369）不会认为它们重复。
后果：升级当日**定时轮与手动轮各跑一次**，双倍消耗 GitHub 配额
（GITHUB_TOKEN 未配置时核心配额仅 60 次/小时，见 worker/scheduled.ts:36 注释）。
建议：迁移期兼容——新 key 生成函数先回读旧键是否已存在，或一次性写 daily:→discovery:github: 映射；
至少在文档中说明升级当日的双跑风险。

### S5【中】“手动触发”没有任何频控，且绕过既有预算闸门
proposal:31-38 的触发接口没有任何限流引用。
**代码事实**：本仓有完整频控设施 src/server/platform/rate-limit.ts:46-134（9 个动作），
但**全部是给普通用户的**，admin 路由普遍不限额（现有 admin/discovery 也不限）。
后果：admin 凭据泄露或误用循环脚本时可无限次排入发现任务（每次都是新 requestKey），
叠加 S1 的键设计，会持续烧 GitHub 配额并产生大量任务行。
建议：即便对 admin 也应加一层上限（如每管理员每小时 N 次手动触发），
或依赖 S3 的枚举 + 每源每天 1 次的硬约束把上界卡死。

## 完整清单（其余）

### S6【中】payload 里的 source/date 未校验，handler 会静默使用默认值
proposal:24 payload: { source, date }。worker 侧 handlers.ts:288-314 只读
run_key/query/search_pages/per_page/max_candidates/max_plugins/max_requests，**不读 source**；
且 handlers.ts:292 在 payload.run_key 缺失时**回落到 discoveryDailyKey(nowMs())**。
后果：若新代码只写 payload 不写 run_key（proposal:24 确实没写 run_key），handler 会用旧键做 runKey，
审计记录（handlers.ts:328 targetId=runKey）与实际去重键不一致。
建议：payload 必须带 run_key 且与 deduplicationKey 同值（现有两处调用都这么做：scheduled.ts:113、
admin/discovery/route.ts:116）；source 若 handler 不用就不要写进 payload。

### S7【中】requestedBy: null 让手动触发失去归责线索
proposal:23 把 requestedBy 写死为 null。现有定时轮传 null 是**对的**
（scheduled.ts:107，系统排入，审计走 actorKind='system'）。
但手动触发若也传 null，background_tasks.requested_by（0001:350）就无法区分“谁点的”；
而审计 operation_records 的 operationKey 是 discovery:<taskId>:<nowMs()>（handlers.ts:324），也不含操作人。
建议：手动轮传 ctx.userId（现有 admin/discovery/route.ts:114 就是这么做的），并额外记一条 admin 操作审计。

### S8【低】没有对 date 的时区/格式约束
proposal:22-24 用 ymd(now)。现有 discoveryDailyKey 明确按站点展示时区（Asia/Shanghai）算，
并在 discovery.ts:121 注释说明“避免服务器时区一变就跨两轮”。提案未说明 ymd 的时区口径。
建议：明确复用 discoveryDailyKey 的 formatDate 口径，不要引入第二个日期函数。

### S9【低】重试常量引用准确，但语义需写明
proposal:41 引用 RETRY_BACKOFF_MS = [30s,120s]、MAX_RECOVERABLE_ATTEMPTS = 3 —— 数值**准确**
（tasks/constants.ts:32,39）。但发现任务重型（handler deadline 300s，constants.ts:45），
3 次重试会反复打 GitHub API，正是 proposal:5 抱怨的“重复调用 GitHub API”的另一种形态。
建议：文档写明“重试属于同一任务的重跑，不产生新任务”——现有队列已保证（finish 走 retry_wait，不新建行），
避免实现者自己加重试循环。

### S10【低】“无新表”正确，注意别顺手引入迁移
proposal:10「不新建表，直接在 background_tasks 上做」——**核实为真**，
task_type 已含 plugin_discovery（field-rules.ts:94、0007 迁移）。
若采纳 S3 的枚举与 S1 的键格式变更，**无需**新迁移；只有引入新索引或 CHECK 才需要。
建议：保持无迁移，把约束放在服务层。

## 一句结论
方案要解决的问题（同一天重复触发导致重复调 GitHub API）**真实存在**，
但 proposal:38 对“幂等”机制的错误描述会直接导致实现者删掉现有正确的 manual:<requestKey> 设计，
把“能重复触发”变成“触发后静默不生效”；同时新路由重复造轮子、source 无枚举、手动入口无频控。
建议：**不新建路由**、**保留 requestKey 语义**、**source 收敛为枚举**、**明确升级日双跑风险**。