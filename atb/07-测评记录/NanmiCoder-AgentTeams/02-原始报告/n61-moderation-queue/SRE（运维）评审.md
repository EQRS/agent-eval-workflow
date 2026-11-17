# n61-moderation-queue SRE（运维）评审

> t51 · n46-sre · 2026-11-17

---

N61（新增审核员角色 + 待审分配）SRE 评审完成：结论「有阻塞项（4 P0 + 3 P1），不建议按现行方案上线」。报告：_work/评审-N61-审核队列分配-SRE.md

关键发现（方案的背景只对了一半）：user_roles.role_type 里"reviewer（测评员）"身份早就存在（0040_user_roles.sql:26），却从未接进审核链路——grep reviewer src/server/modules/moderation 命中 0，人工决定的唯一闸门是 isAdmin（moderation/service.ts:465-468）。所以方案是在正确的方向上绕路：不是"新增角色分担压力"，而是把既有 reviewer 接通。

P0-1 §1 的 ALTER 在 SQLite 是语法错误：SQLite 不支持 DROP/ADD CONSTRAINT，迁移直接失败 → compose.yaml:27-42 的 migrate 前置容器失败 → web/worker 因 depends_on: service_completed_successfully 永不启动 = 全站宕机。正确做法只有重建表（0007 是范例），而 users 被全库几乎所有表引用且属 REQUIRED_TABLES（migrate.ts:142-143），重建窗口 PRAGMA foreign_keys=OFF 期间 web/worker 两个常驻进程仍在写（compose.yaml:44-108）；migrate.ts:97-115 明说这类迁移不替回滚，半成品只能靠 restore-backup.mjs 恢复到 14 天前（backup.service:26）。另外 0043 与 N42 撞号，migrate.ts:51-55 无重号检测。
P0-2 §1/§3 新造 moderator 与既有设计冲突：0040:5-9 明写 users.role 是权限底座（user/admin，决定能否进后台），测评员/开发者是叠加的能力标记，"混进同一个 CHECK 会让『管理员是不是测评员』没有答案"。方案没回答"管理员是不是审核员"。若坚持新增，还会牵动 CallerContext.role（context.ts:10-15）与 17 个文件里的 isAdmin/assertCanAdmin（实测 60+ 处），任何 role !== 'admin' 的分支都会把 moderator 静默降级。建议改用 user_roles('reviewer','active') + canModerate，零迁移。
P0-3 §3"每 5 分钟扫描"没有落点：仓库只有 worker 主循环（main.ts:170-206，节拍 constants.ts:48 的 10s）与两个 systemd timer；做成 setInterval 违反 rate-limit.ts:182-184 的明确立场（"要进程能被干净关掉"，compose.yaml:69,94 只有 15s/30s 停机窗口），做成新 timer 又是运维口袋里的隐式依赖。建议做成 worker 主循环里的幂等 tick（同 ensureDailyDiscoveryTask，main.ts:178-188 + scheduled.ts:28 的 60s 节流）。
P0-4 分配表与逻辑：无 UNIQUE(submission_id) 且查询不排除已分配 → 每 5 分钟重复插入；reviewers 为空时 reviewers[NaN] 抛 TypeError，被 main.ts:211-218 吞成日志 → 静默失效；无 TTL/回收，审核员被禁用后分配永久卡住（制造新积压）；无审计（应走 recordOperationInTx，operations/service.ts:161-174）；i % length 不看在手量，负载均衡是假的。

P1-5 积压无可观测：diagnose.mjs:136-175 只统计 background_tasks，health.ts:1-9 不做业务统计 → 改造前后的"积压"无法量化，分配器静默失效无告警。建议加三项判据并让退出码 1。
P1-6 访问控制：让审核员进 /admin/submissions 等于给全量数据面——admin/submissions/route.ts:20-72 无 assignee 过滤，decisions/route.ts:21-73 不校验归属；且同前缀下还有用户管理/备份/设置。建议独立 /api/moderation/* + 校验 review_assignments.reviewer_id；另外 moderation/service.ts:497 写死 actorKind:'admin'，审核员的决定会被记成管理员做的。
P1-7 撤销与在职状态：users.role 是单值列无法表达"既是审核员又是管理员"，撤销后历史分配仍指着他；review_assignments 缺 status/completed_at。建议复用 user_roles（撤销自带 revoked_by/revoked_at）并给分配表补状态与索引。

另附 9 项上线验收清单。