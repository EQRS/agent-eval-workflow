# N46「插件归属争议的申诉流程」方案 · SRE（运维）评审

> 评审对象：`atb/tasks/n46-ownership-dispute/environment/proposal.md`（30 行）
> 评审身份：SRE（运维：队列治理 / 审计与可追溯 / 变更与回滚 / 容量与滥用）
> 事实基线：`migrations/0040_user_roles.sql`、`migrations/0042_audit_immutable.sql`、`src/server/modules/roles/{ownership,service}.ts`、`src/server/modules/identity/admin.ts`、`src/server/modules/operations/service.ts`、`src/server/platform/{migrate,health,rate-limit}.ts`、`src/server/platform/tx.ts`

## 0. 结论

**不建议按现方案实施。** 这份方案在运维上有三个「上线即出事」的形态：

1. **第 26 行「管理员直接改 plugin_ownerships 的归属人」会撞库里的部分唯一索引**（`migrations/0040_user_roles.sql:119-120` 的 `uq_plugin_ownerships_verified`）——改归属这个动作**在数据库层就不成立**，必须先撤销旧的再建新的，还要写审计。仓库已有既定实现（`roles/ownership.ts:313-367`），方案绕开了它。
2. **这个功能会生产两类工单**（归属争议 + 认领复核），而方案没有任何队列治理：没有状态机、没有查重、没有频控。
3. **归属变更没有审计**：`operation_records` 是全站唯一审计载体（`0042` 刚加固过），方案第 26 行改完就完事，事后无法回答「谁、在什么时候、凭什么把插件从 A 划给了 B」——而这恰恰是争议场景下**唯一需要查证的东西**。

一句话：**方案把「争议处理」写成了「一个表 + 一个 UPDATE」，但这件事在运维上的全部价值都在「过程可追溯」上。**

---

## 1. 第 26 行「直接改归属人」在数据库层不成立 —— proposal 第 25–26 行

**① 问题**：方案第 26 行「管理员查看证据后，直接改 `plugin_ownerships` 的归属人」。

**② 后果（核实依据）**：
- `migrations/0040_user_roles.sql:119-120`：`CREATE UNIQUE INDEX uq_plugin_ownerships_verified ON plugin_ownerships(plugin_id) WHERE status = 'verified';` —— **一个插件只能有一个 verified 归属人**。直接把 B 那行改成 verified 会撞索引，而 `errors-http.ts:120-126` 只把 `DomainError` 映射成业务错误，**`SQLITE_CONSTRAINT` 会变成 500**。
- 这条 500 在本仓**已经被真实踩过并写在注释里**：`roles/ownership.ts:239-250`——「审查复现过一条 500：A 被驳回 → B 自动 verified → A 重新绑 GitHub 再认领……直接 UPDATE 成 verified 就撞了部分唯一索引，errors-http 只映射 DomainError → 500」。也就是说，**方案第 26 行要求管理员手工复现这个已知的 500**。
- 正确做法仓库里也有：`roles/ownership.ts:327-337`——**先查 taken 并给出人话**（`:329-332`「这个插件已经有归属人了，请先处理那一条」），再改状态。方案把这个防护全部跳过了。
- 还有一层：改归属会让两人的**开发者资格**立刻变化——`roles/service.ts:444` 与 `ownership.ts:369-376` 的 `countVerifiedForUser` 是开发者门槛的判定依据；A 的插件划给 B，A 可能立刻不再满足资格，但 `user_roles` 里的 developer 行**不会自动撤销**（`service.ts:738-760` 的 `revokeRole` 是独立动作）。于是出现「已是开发者但一个已验证插件都没有」的悬空状态，而它没有任何监控。

**③ 建议**
- 方案第 3 节改为「**走 `pluginOwnershipService` 的既定转移路径**」：撤销旧的 verified（写 `reviewer_id`/`reviewed_at`/理由）→ 建立新的 → 两者在**同一事务**内完成。
- 若确实要直改，必须先 `SELECT ... WHERE plugin_id = ? AND status='verified'` 并显式处理冲突，**绝不允许裸 UPDATE**。
- 明确「换人之后原归属人的 developer 身份怎么办」：自动降级 / 保留但标记待复核 / 通知。三者必须选一个并写进方案——**它决定了下线后的工单形态**。

---

## 2. 归属变更没有审计：争议场景下最该留下的东西没有 —— proposal 第 25–26 行

**① 问题**：方案第 26 行没有任何审计写入。

**② 后果（核实依据）**：
- `operation_records` 是全站审计与幂等去重的**唯一载体**（`operations/service.ts:6`、`identity/admin.ts:23`），且 `migrations/0042_audit_immutable.sql:17-27` 刚用 `BEFORE DELETE`/`BEFORE UPDATE` 两个触发器把它**锁成不可删改**；头注释（`:5-12`）写明原因：「删除审计行即可让已消费的 operation_key 重新可用」是一条真实的伪造通道。
- 本仓的既有惯例是**每一次真实的管理员变更都在同一事务内写一条 operation_records**（`identity/admin.ts:23-25` 的头注释原文、`:256-293` 的实现），动作名统一用命名空间前缀（`:53-54`）。
- 而 `plugin_ownerships` 的现有流程**同样没有写审计**：`ownership.ts:313-367` 的 `reviewClaim` 只改状态 + 发通知，**没有 `recordOperationInTx`**。这不是方案引入的，但它意味着：**方案把「管理员直接改归属」做成日常操作之后，站点将第一次出现「关键权限变更完全无审计」的状态**，而这正好发生在争议场景——事后复盘要回答的就是「谁改的、凭什么改」。

**③ 建议**
- 按 `identity/admin.ts:47-54` 的命名惯例定义 action_type：`ownership.dispute_open` / `ownership.dispute_resolve` / `ownership.transfer`，**全部经 `recordOperationInTx` 落在同一事务**。
- **顺带补上现有缺口**：给 `reviewClaim` 也加审计（`ownership.claim_verify` / `ownership.claim_reject`），`operation_key` 带上 `reviewedAt`（对齐 `ownership.ts:182-197` 里「同一行会被反复使用」的既有结论）。
- 运维验收：任意一次归属变更后，能用一条 SQL 回答「这个插件历史上被谁、在什么时候、从谁划给谁」。

---

## 3. status 是自由文本：处理流程无法依赖它，且与库内既有约束口径不一致 —— proposal 第 17 行

**① 问题**：`status TEXT NOT NULL DEFAULT 'pending'`，**没有 CHECK、没有枚举**。

**② 后果**：
- 本仓同类表**全部带 CHECK**：`user_roles.status IN ('active','revoked')`（`0040:27`）、`role_applications.status IN ('pending','approved','rejected','withdrawn')`（`0040:52-53`）、`plugin_ownerships.status IN ('pending','verified','rejected')`（`0040:87-88`）。方案这一张**例外**，写进任何一个错字（例如 pendign）都不会报错。
- 后果直接落在运维上：**脏状态行会永远留在 pending 里**，管理员看不到、也处理不掉（没有状态能让它离开）。这类行只能靠手工 UPDATE 清理。
- 方案也没有定义**状态迁移**：至少需要 `resolved_kept`（维持原归属）与 `resolved_transferred`（改属）两种终态，否则「处理完了」和「还挂着」无法区分。

**③ 建议**：`status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','resolved_kept','resolved_transferred','rejected'))`，并按 `0040:95-110` 的方式补成对约束（例如 resolved_* 必须有 `resolved_by` / `resolved_at`，pending 不许带处理痕迹）。**照抄 `plugin_ownerships` 的 CHECK 写法即可**——它在注释里已经把两个坑写清楚了（`:98-109`）。

---

## 4. 「任何用户都能对任意插件提争议」= 一个没有配额的工单入口 —— proposal 第 22–23 行

**① 问题**：方案第 23 行的文案没有任何前置条件。

**② 后果（运维视角）**：
- **工单量没有任何上界**：插件数 × 用户数就是最大工单量。这是**唯一一个「用户数 × 对象数」规模**的入口，比评论/举报的规模都大。
- 本仓对写入口的既有纪律是「**频控必须挂在昂贵动作之前**」（`rate-limit.ts:190-195` 的注释原文：「必须在真正做昂贵/写库的动作之前调用」），而动作枚举 `RateLimitAction`（`rate-limit.ts:46-55`）里**没有**「争议」这一类——方案要新增就必须加一条规则，否则这个入口是全站唯一不受限的用户写操作。
- 争议入口天然会被用来**做信息探测**：提交成功/失败、「这个插件已有归属人」这类差异，会让攻击者以极低成本枚举「哪些插件其实没人认领」。方案没有定义**对外错误语义**。

**③ 建议**
- 新增一条 `RateLimitAction`（例如 `ownership_dispute`，建议每用户每天 3 次，对齐 `forum_board_application` / `role_application` 的档位 `rate-limit.ts:118-133`）。
- **前置校验**：只有当插件当前**有 verified 归属人**时才允许提争议（没有归属人应该走「认领」，不是「争议」）——这一条就能把工单量砍掉一大截，并顺手消掉「枚举无主插件」的探测面。
- 对外统一返回「已受理」，**不透露**当前归属人是谁、也不透露是否已有其他争议。

---

## 5. 没有查重：同一人可反复提交同一争议，撑爆表并淹没真实工单 —— proposal 第 12–19 行

**① 问题**：表上没有 `UNIQUE(plugin_id, claimant_id)`，也没有「同一插件同时只允许一条 pending 争议」的约束。

**② 后果**：
- 本仓对「同一主体 + 同一对象同时只允许一条待审」**有明确范式**：`uq_role_applications_pending ON role_applications(user_id, role_type) WHERE status = 'pending'`（`0040:69-70`，注释在 `:67-68` 说明了「被驳回后能再申请」靠状态放行）。方案没有对应约束。
- 没有约束时，**管理员队列会被同一份争议刷屏**，真实争议被淹没——这正是「运维看不到信号」的典型形态，而且它是**无成本**的：一次点击就多一行。

**③ 建议**：`CREATE UNIQUE INDEX uq_ownership_disputes_pending ON ownership_disputes(plugin_id, claimant_id) WHERE status = 'pending';`（部分唯一索引，照 `0040:69-70`）。另加 `CREATE INDEX idx_ownership_disputes_queue ON ownership_disputes(status, created_at, id)`——对齐 `0040:115` 的队列索引形态，否则管理员队列会全表扫。

---

## 6. 迁移与自检：新表没有落任何部署产物 —— proposal 第 11–19 行

**① 问题**：方案只写了一句 `-- migrations/0043_ownership_disputes.sql`，没有登记、没有回滚口径。

**② 后果**：
- 本仓的权威结构是 `migrations/*.sql`（`migrate.ts:6`），执行者是 `migrate` 一次性进程（`docker/compose.yaml:27-42`，web/worker 等它 `service_completed_successfully`）。没有迁移文件就没有表。
- 新表**未登记进 `REQUIRED_TABLES`**（`migrate.ts:142-237`）→ `health.ts:76-82` 的 `db.required_tables` 检查覆盖不到它 → 「表建了但迁移半途失败」不会引发 503，**而是在第一次有人提争议时炸**。这正是 0037 那批表的教训（`migrate.ts:214-215` 的注释：「这 10 张此前没登记，自检报'缺表'却什么都不缺，等于把这道门禁废掉了」）。
- 编号 `0043` 正确（现有最后一个是 `0042`；`migrate.ts:51-55` 用 `localeCompare(b,'en')` 排序）。
- 格式：`0040_user_roles.sql:15` 显式写了 `PRAGMA foreign_keys = ON;`（因为该迁移会 DROP 索引重建）。新迁移建表**不需要**关外键，也不该带 `dsh:migration-transaction=off`（`migrate.ts:30-49`）——方案应写明这一点，避免后来人照抄 0040 的 PRAGMA 引入不必要的行为。

**③ 建议**：交付物清单写死——迁移文件、`REQUIRED_TABLES` 追加、审计 action_type 常量、频控规则、后台队列入口；回滚口径写「`DROP TABLE ownership_disputes` 不影响既有归属数据」（对齐 `docs/docker约定.md` §6「迁移是向前的」）。

---

## 7. 归属变更的连锁效应没有定义：开发者资格、通知、事务边界 —— proposal 第 25–26 行

**① 问题**：方案第 26 行把「改归属」当成一个孤立动作。

**② 后果（核实依据）**：
- **开发者资格**：`roles/service.ts:444` 与 `ownership.ts:369-376` 用同一条 SQL 数 verified 归属；注释（`service.ts:429-430`）明确写了「开发者一档与 plugin_ownerships 的 countVerifiedForUser 用逐字相同的 SQL」。改归属会改变这个计数，但 `user_roles` **不会自动撤销**（`service.ts:738-760`），且 `0040:27` 的 CHECK 要求 `revoked_by`/`revoked_at` 成对——手工补状态很容易写错。
- **通知**：现有认领流程在审核事务内投递通知（`ownership.ts:173-200`，去重键带 `reviewedAt`，`:182-188` 说明了为什么必须带轮次）。争议处理**同样会改变双方用户的状态**，方案没有通知设计——下线后会出现「我的插件不见了但没人告诉我」。
- **事务边界**：「撤销旧 + 建立新 + 写审计 + 发通知」必须在**一个事务**内（对齐 `ownership.ts:173` 的「在审核事务内投递」与 `tx.ts:42-52` 的同步事务纪律）。方案没有提。

**③ 建议**：在方案里补一张「争议处理的事务清单」：`撤销旧归属 → 建立新归属 → 写 operation_records → 投递两条通知（原归属人 + 新归属人） → 更新争议行状态`，全部同事务；并明确「developer 身份是否随归属撤销而撤销」的产品决策。

---

## 8. 验收：方案第 28–29 行「争议有据可查 / 归属可纠正」不可证伪 —— proposal 第 28–29 行

**① 问题**：两句话都是定性描述。**② 后果**：无法判断是否做到；更糟的是「归属可纠正」在方案里其实是**不可纠正**的（§1 的 500）。

**③ 建议**（建议直接写进方案）：

| # | 断言 | 场景 |
|---|---|---|
| 1 | 插件已有 verified 归属人时执行争议转移 → **不是 500**，而是明确的业务结果 | 复现 `ownership.ts:239-250` 记录的那条 500 |
| 2 | 转移后 `SELECT * FROM plugin_ownerships WHERE plugin_id=? AND status='verified'` 恰好 1 行 | 唯一索引兜底 |
| 3 | 每次转移在 `operation_records` 里**恰好 1 条**，含 actor / 时间 / 旧新归属人 | 审计可追溯 |
| 4 | 同一用户对同一插件重复提交 → `uq_ownership_disputes_pending` 命中，返回「已受理」而非新行 | 查重 |
| 5 | 单用户第 4 次提交当天争议 → 429 | 频控 |
| 6 | 迁移后 `/api/health` 的 `db.required_tables` 含 `ownership_disputes` | 自检门禁 |

## 9. 一句话给作者

仓库在归属这件事上**已经有全套基础设施**：部分唯一索引兜底（`0040:119-120`）、既定的审核与转移路径（`ownership.ts:313-367`）、不可删改的审计（`0042`）、成对的 CHECK 约束（`0040:95-110`）、部分唯一查重范式（`0040:69-70`）。**方案真正需要新建的只有一张「争议表」和它的状态机与队列索引**——其余部分应当接上已有基础设施，而不是在 SQL 里「直接改」。
