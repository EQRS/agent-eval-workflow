# 评审：新增审核员角色并分配待审任务 · 性能工程师视角

> 评审对象：`atb/tasks/n61-moderation-queue/environment/proposal.md`（49 行）
> 评审人身份：**性能工程师**（只看扫描频率与查询计划、写放大、索引、队列公平性与吞吐、worker 并发位；不评权限模型）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**§3 的「每 5 分钟全表扫 + 无去重插入」在现有索引下会持续产生重复行，而重复行会让「待审积压」这个指标本身失真**——
> 越积压，扫描越重，重复越多，形成正反馈。方案没有一条数量级界定（扫多少、写多少、保留多久）。

---

## 摘要（前 5 条最重要）

1. 【严重】**§3 的 `SELECT * FROM content_submissions WHERE processing_state='manual_review' LIMIT 50` 没有 `ORDER BY`，也没有排除已分配——每 5 分钟都会重复插入同一批。**
   依据：`proposal.md:31-41`（查询 → `forEach` → 无条件 `INSERT`）；
   仓库约束：`migrations/0001_baseline.sql:338` 只有 `idx_submissions_state(processing_state, submitted_at, id)`，**没有 `(processing_state, id)`**；
   而 `LIMIT 50` 没有 `ORDER BY` 时 SQLite 只能按某个扫描顺序给出「前 50 条」——用现有索引即**按 `submitted_at` 升序**，
   而这恰好是「最老的那 50 条」，也恰好是**最容易被重复选中**的那 50 条。
   后果：每 5 分钟固定产生最多 `50 × reviewers.length` 行新插入？不——是**每条待审固定再插 1 行**，
   即 12 次/小时 × 24 = **288 倍于待审数的写入/天**。一个 50 条待审的队列，一天写出 14,400 行分配记录。
   建议：① 查询加 `ORDER BY submitted_at ASC, id ASC`；② `review_assignments` 加 `UNIQUE(submission_id)` 或 `UNIQUE(submission_id, reviewer_id)`
   并用 `INSERT ... ON CONFLICT DO NOTHING`；③ 更根本地：用 `LEFT JOIN review_assignments ... WHERE ra.id IS NULL` 只取未分配的。

2. 【严重】「每 5 分钟全表扫」与现有队列查询叠加在同一张最热的表上。
   依据：`proposal.md:28`；`src/app/api/admin/submissions/route.ts:33-43`（后台待办列表：`WHERE s.processing_state = ?` + `ORDER BY s.submitted_at ASC, s.id ASC`，**无 LIMIT**）；
   `0001_baseline.sql:338` 的索引是 `(processing_state, submitted_at, id)`——`admin` 那条查询能吃到索引，**但结果集是全量的**；
   方案的扫描则无法用 `LIMIT 50` 真正截断（没有 `ORDER BY` 时 SQLite 可以早停，但一旦按上面建议加 `ORDER BY` 就必须走索引）。
   后果：`content_submissions` 是全站提交表，`processing_state` 的变化频率与用户提交量同阶；
   `better-sqlite3` 是同步驱动（`src/server/platform/db.ts:6-8`），扫描跑在 worker 主循环里会挤占任务处理时间。
   建议：把扫描改成「只查未分配」的带索引查询（见下），并把频率从 5 分钟改成「事件驱动 + 兜底扫描」（提交转 `manual_review` 时立即分配，定时只做兜底）。

3. 【严重】分配表的写入没有事务、没有批量语句复用，`forEach` 里逐条 `db.prepare(...).run(...)`。
   依据：`proposal.md:37-41`（循环内 `db.prepare` 每次重新编译语句）；`src/server/platform/db.ts:52-56`（WAL + `synchronous = FULL`，**每次提交一次 fsync**）；
   `src/server/platform/tx.ts:1-9`（「跨模块流程创建一次短事务」「不得各自提交独立事务后假装整体原子成功」）。
   后果：50 条待审 = 50 次 `prepare` + 50 次独立提交 = **50 次 fsync**；在云盘/机械盘上可达数百毫秒到数秒，
   且这 50 次提交是逐条可见的（没有事务包裹），中途崩溃会留下**半分配的队列**。
   建议：`prepare` 提到循环外 + 整个批次一个 `withTransaction`；配合 `ON CONFLICT DO NOTHING` 让重跑安全。

4. 【高】没有处理「审核员为空」与「审核员被禁用」——两种情况下分配循环要么抛错、要么把任务分配给不会再出现的人。
   依据：`proposal.md:34-41`（`reviewers[i % reviewers.length]`）；
   `src/server/platform/context.ts:13`（`role: 'user' | 'admin' | null`——**当前根本没有 moderator 这个取值**，见 P-05）；
   `src/app/api/admin/submissions/route.ts:24`（后台入口 `assertCanAdmin`）。
   后果：`reviewers.length === 0` 时 `i % 0 === NaN` → `reviewers[NaN]` 为 `undefined` → `.id` 抛 TypeError，
   整个扫描 tick 崩溃（若在 worker 里，这一轮的任务处理也被打断）；
   已被禁用（`account_status='disabled'`）的审核员名下的任务会**永久滞留**，没有任何回收路径——积压指标只增不减。
   建议：无审核员时**跳过本轮并记日志**（不要排空队列）；分配查询里 `AND account_status = 'active'`；
   加一个「重新入池」路径：非 active 审核员名下的分配记录定期回到未分配状态。

5. 【高】方案默认「审核员通过 `/admin/submissions` 处理」，但那个接口**没有分页**，全员待审 + 每页全量返回。
   依据：`src/app/api/admin/submissions/route.ts:33-43`（`WHERE processing_state = ?` 无 `LIMIT`、无分页参数，`:65` 的 `total: rows.length` 说明它返回全量）；
   `src/app/admin/page.tsx:36,43`（后台首页 `TODO_STATES` 固定列举三个状态）。
   后果：积压从「管理员看不过来」变成「审核员也看不过来」时，**这条无 LIMIT 的查询会返回全部待审行**，
   在 pending 多的时候直接是一次大结果集序列化 + 传输；这也放大了摘要 2 的扫描成本。
   建议：分配后审核员只看「分配给自己的」，并按 `(reviewer_id, assigned_at)` 建索引 + 游标分页；
   后台列表同步加 `LIMIT`。

---

## 完整问题清单（按严重度）

### P-01 【严重】扫描查询无 ORDER BY、无排除已分配 → 重复插入
- 现场：`proposal.md:31-41`。
- 依据：`0001_baseline.sql:338`（`idx_submissions_state(processing_state, submitted_at, id)`）；方案没有 `UNIQUE(submission_id)`（`proposal.md:19-24`）。
- 后果：见摘要 1。具体量级：12 次/小时扫描，每次对「最老且未完成」的 50 条各插 1 行 ⇒ 一天 14,400 行（50 条待审时），表体积与查询成本线性上涨。
- 建议：`ORDER BY submitted_at ASC, id ASC`；`UNIQUE(submission_id)`；`INSERT ... ON CONFLICT DO NOTHING`；或直接 `LEFT JOIN ... IS NULL` 只取未分配。

### P-02 【严重】扫描频率与现有无 LIMIT 的后台查询叠加
- 现场：`proposal.md:28`。
- 依据：`src/app/api/admin/submissions/route.ts:33-43,65`；`db.ts:6-8`（同步阻塞）。
- 后果：见摘要 2。
- 建议：事件驱动分配 + 兜底扫描；后台列表加 LIMIT。

### P-03 【严重】无事务、无语句复用 → 每行一次 fsync
- 现场：`proposal.md:37-41`。
- 依据：`db.ts:52-56`（`synchronous = FULL`）；`tx.ts:1-9`。
- 后果：见摘要 3。
- 建议：预编译 + 单事务批量。

### P-04 【高】审核员为空 / 被禁用时的行为未定义
- 现场：`proposal.md:34-41`。
- 依据：`proposal.md:40`（`i % reviewers.length`）。
- 后果：NaN 索引崩溃；禁用审核员名下任务永久滞留。
- 建议：空集合跳过；只分配给 active；加回收路径。

### P-05 【高】`users.role` 的取值集合在代码里是**封闭类型**，方案只改了 SQL 侧
- 现场：`proposal.md:9-15`。
- 依据：`src/server/platform/context.ts:13`（`role: 'user' | 'admin' | null`）、`:24`（`isAdmin` 只认 `'admin'`）、`:48-52`（`assertCanAdmin` 断言 `role: 'admin'`）；
  `src/app/api/admin/submissions/route.ts:24`（后台入口用 `assertCanAdmin`）。
- 后果（性能侧）：方案说审核员走 `/admin/submissions`——该路由的读路径会**先被权限挡掉**，于是「分配了但没人能看」；
  为了打通，实现者很可能在 `assertCanAdmin` 或页面里加 `role === 'moderator'` 的短路分支，
  **于是后台那条无 LIMIT 的全量查询会被更多账号调用**，把 P-02 的成本乘以活跃审核员数。
- 建议：权限放开必须与「按分配过滤 + 分页」同时做，不能只放开入口。

### P-06 【中】分配表没有索引，但读取路径一定会按 `reviewer_id` 过滤
- 现场：`proposal.md:19-24`（只有 PK）。
- 依据：仓库同类表的范式：`0001_baseline.sql:339`（`idx_submissions_author`）、`0040_user_roles.sql:64-65`（`idx_role_applications_user`、`idx_role_applications_queue`）。
- 后果：审核员打开「我的待审」= 全表扫 `review_assignments`；而该表按 P-01 的速度增长，很快就不能全扫。
- 建议：`CREATE INDEX idx_assignments_reviewer ON review_assignments(reviewer_id, assigned_at, id);` + `UNIQUE(submission_id)`。

### P-07 【中】没有分配结果的保留期；表只增不减
- 现场：`proposal.md` 无 TTL。
- 依据：`content_submissions` 的历史是**不可变快照**（`0001_baseline.sql:333-336` 的唯一索引 + `0002` 的不可变触发器族），所以分配表会跟着历史一起长。
- 后果：一年后 `review_assignments` 的行数 = 历史待审数 × 重复倍数（见 P-01）。
- 建议：要么给 `UNIQUE(submission_id)` 让「一个提交一条分配」（天然有界），要么加 `resolved_at` 与定期归档。

### P-08 【低】无「积压深度 / 平均等待时长」指标，方案的收益无法验收
- 现场：`proposal.md:47-49`。
- 依据：现有可观测手段是后台页面与 `operation_records`（`0001:411-431`）。
- 后果：无法回答「分配机制是否真的减少了积压」。
- 建议：按 `submitted_at` 与 `assigned_at` 的差值统计 P50/P95 等待时长（一条聚合查询 + 索引即可）。

---

## 结论与建议

**方案在性能上会形成正反馈恶化**：积压越多 → 每次扫描扫得越多 → 重复插入越多 → 表越大 → 查询越慢 → 积压感知更久。
而它的三条核心语句都缺少最基本的数量级界定（去重、事务、边界条件）。

**建议的最小改造**：
1. `review_assignments`：`UNIQUE(submission_id)` + `idx_assignments_reviewer(reviewer_id, assigned_at, id)`。
2. 分配改为「只取未分配」：`... LEFT JOIN review_assignments ra ON ra.submission_id = s.id WHERE s.processing_state='manual_review' AND ra.id IS NULL ORDER BY s.submitted_at ASC, s.id ASC LIMIT 50`。
3. 预编译语句 + 整批一个 `withTransaction`；`ON CONFLICT DO NOTHING` 让重跑安全。
4. 触发方式：提交转 `manual_review` 时**事件驱动**分配，5 分钟扫描只做兜底（或拉到 30 分钟）。
5. 后台入口按「分配给我的」过滤并分页；`/api/admin/submissions` 同步加 `LIMIT`。
6. 指标：积压深度、等待时长 P50/P95。

> 评审人声明：本报告只覆盖性能/容量/索引/吞吐视角。`users.role` 的 CHECK 无法用 `ALTER TABLE ... DROP CONSTRAINT` 修改（SQLite 不支持，必须重建表）、
> 以及 moderator 角色如何与既有权限判定共存，属于迁移与安全视角，本文不代替它们的结论；
> 但 P-05 的结论在性能侧同样成立：**放开权限入口而不加过滤与分页，会把无 LIMIT 查询的成本乘以审核员人数**。

> 前提核实：`content_submissions.processing_state` 含 `manual_review` 取值（`0001_baseline.sql:315-316`）真实存在；
> `idx_submissions_state`（`:338`）真实存在；后台待办查询真实存在且确无 `LIMIT`（`src/app/api/admin/submissions/route.ts:33-43`）；
> 但 `users.role` 在代码侧是 `'user' | 'admin' | null` 的**封闭类型**（`context.ts:13`），方案只给了 SQL 侧改动。
