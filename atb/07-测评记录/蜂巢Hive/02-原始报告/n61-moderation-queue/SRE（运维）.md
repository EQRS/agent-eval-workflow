# n61-moderation-queue · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0028
> 模型 deepseek-chat · 7518 字 · 23 次工具调用

---

证据已足够。以下评审**只**从 SRE 视角出发。

---

## 一、迁移方案本身跑不起来（P0：上线即失败，影响所有环境）

**① 问题**
`proposal.md:9` 的
```sql
ALTER TABLE users DROP CONSTRAINT users_role_check;
```
**SQLite 根本不支持 `DROP CONSTRAINT`**，也从来没有过 `ALTER TABLE ... ADD CONSTRAINT`。这是 Oracle/PostgreSQL 语法。而且 `role` 的 CHECK 是**匿名/行内**的（`0001_baseline.sql:29`：`role TEXT NOT NULL CHECK (role IN ('user','admin'))`），库中不存在名为 `users_role_check` 的约束对象。

**② 后果（文件:行号）**
- `proposal.md:9-12` → 迁移执行到这一行直接抛 `near "CONSTRAINT": syntax error`。
- `src/server/platform/migrate.ts:119-122`：执行器对**无** `dsh:migration-transaction=off` 标记的文件用 `db.transaction()` 包裹。DDL 抛出后整文件回滚，`0003` 之后的 `review_assignments` 建表也一起不执行。
- 更关键：**启动不会失败**。`src/instrumentation-node.ts:33-41` 的 `selfCheck` 只把问题 `log.error` 出来，注释明确写了「刻意不在这里退出进程」。结果是服务以**旧 schema + 缺失表**的状态对外服务，`/api/health` 才报 503——这就是典型的「进程活着、功能全坏、告警滞后」的故障形态。

**③ 建议**
- SQLite 改 CHECK 的唯一路径是**重建表**（新建 → `INSERT ... SELECT` → `DROP` → `RENAME`），且因 `users` 被 20+ 张表外键引用（`0001` 全篇都是 `REFERENCES users(id)`），必须照 `0007_task_type_plugin_discovery.sql:24` 和 `0041_role_notifications.sql:34` 的手法：文件头声明 `-- dsh:migration-transaction=off` + `PRAGMA foreign_keys = OFF` + 自管 `BEGIN/COMMIT` + 结尾 `PRAGMA foreign_keys = ON` 与 `PRAGMA foreign_key_check`。
- 并在 `migrate.ts:182` 的 `REQUIRED_TABLES` 里补 `review_assignments`，让漏建表在启动自检中变成显式问题而不是 500。

**④ 建议（更优）**
按 `0040_user_roles.sql:1-14` 已经确立的口径——**`users.role` 是权限底座，能力用 `user_roles` 表叠加**——审核员根本不需要动 `role` 的 CHECK。`user_roles.role_type` 已经有 `'reviewer'` 取值（`0040:29`）。直接复用，**零 schema 风险**，还省掉一次全库最热表的重建。这条我强烈推荐。

---

## 二、「`0043` 这个编号可用」是持久的合并/回滚风险（P1）

**① 问题**
`proposal.md:9` 写死 `migrations/0043_moderator_role.sql`。但 `0040_user_roles.sql:14-17` 留了一段非常具体的教训：*「0037/0038 被讨论区 PR 占用、0039 被收藏重构占用，都还未合入 main……用 0040 可以让三个 PR 任意顺序合并（0027 已有跳号先例）」*。同一份文件 `0041_role_notifications.sql:20-33` 更直接用「⚠️ 合并顺序是硬约束」警告了 CHECK 并集漂移。

**② 后果**
- `migrations/` 目录当前已到 `0042`。本 PR 一旦与他人并行的 `0043` 冲突，文件名重复会让 `migrate.ts:64` 的 `localeCompare` 排序产生**不确定**的先后；而 `0041:30-32` 已实证过这类后果：后执行者用自己的口径覆盖前者，**`kind` 列表随合并顺序漂移**。
- 运维侧表现：同一个 commit、不同环境（预发先合、生产后合）跑出**不同的 schema**，且 `schema_migrations` 里都显示"已应用"。这是最难排查的一类环境漂移。

**③ 建议**
- 开 PR 时就当面确认编号，合入前 rebase 复核；如果 reviewer 方案要扩充 `notifications.kind`（见下条），务必**不要**再走一次 `notifications` 全表重建，而是与大 PR 统一口径一次性并集。

---

## 三、`review_assignments` 表设计缺失约束，脏数据会交给运维擦屁股（P1）

**① 问题**（`proposal.md:15-21`）
```sql
CREATE TABLE review_assignments (
  id TEXT PRIMARY KEY,
  submission_id TEXT NOT NULL REFERENCES content_submissions(id),
  reviewer_id   TEXT NOT NULL REFERENCES users(id),
  assigned_at   INTEGER NOT NULL
);
```
对照本仓库全部 42 个迁移的既有风格，这张表把约定全丢了：无 `ON UPDATE RESTRICT ON DELETE RESTRICT`、无 `assigned_at >= 0` CHECK、**无任何唯一约束**、无索引、无状态列（分配后是待办/已处理/已超时？）。

**② 后果（文件:行号）**
- `proposal.md:26` 的 `pending.forEach(... INSERT ...)` 是**全量盲插**，没有 `NOT EXISTS` / `ON CONFLICT`。既然每 5 分钟扫一次（`proposal.md:23`），同一条 `submission_id` 每轮都会被重新插一条 → **每 5 分钟产生 50 条重复分配**。一天 = 14,400 条纯垃圾行，且审核员待办列表按此渲染会瞬间爆炸。
- 没有 `UNIQUE(submission_id)`：同一条内容可能同时被分给两个审核员，两人各自提交审核结论 → `content_moderation_records`（`0001:230`）多出一条互相冲突的记录，且其 `UNIQUE(submission_id, operation_key)` 拦不住（不同操作人不同 key）。
- 没有 `submission_id`/`reviewer_id` 索引：审核员查"我的待办"是全表扫，随上述垃圾行线性劣化。
- 缺 `ON DELETE RESTRICT`（对照 `0001:232` 的既有写法）：恶意/误删 `content_submissions` 会留下悬空分配。

**③ 建议**
```sql
CREATE TABLE review_assignments (
  id            TEXT PRIMARY KEY,
  submission_id TEXT NOT NULL REFERENCES content_submissions(id) ON UPDATE RESTRICT ON DELETE RESTRICT,
  reviewer_id   TEXT NOT NULL REFERENCES users(id) ON UPDATE RESTRICT ON DELETE RESTRICT,
  assigned_at   INTEGER NOT NULL CHECK (assigned_at >= 0),
  UNIQUE (submission_id)          -- 一条内容同时只有一个负责人
);
CREATE INDEX idx_review_assignments_reviewer ON review_assignments(reviewer_id, assigned_at DESC);
```

---

## 四、每 5 分钟扫描器在现有 worker 架构里无处安放（P0：静默不跑）

**① 问题**
`proposal.md:23`「每 5 分钟扫描待审队列」——方案完全没说它跑在哪。我核对了 worker：`src/worker/main.ts:104` 的 `scheduledTasks: true` 只接入了 `ensureDailyDiscoveryTask`；`scheduled.ts:4-16` 明确写「当前只有『每日插件目录自动发现』」，且排程机制是**按天幂等 dedup key**（`scheduled.ts:7-10`）。

**② 后果**
- 若要作为 `background_tasks` 排入，必须扩 `background_tasks.task_type` 的 CHECK（`0007_task_type_plugin_discovery.sql:29`，闭集为 `'review_check','review_publish','comment_check','comment_publish','plugin_discovery'`）——**又是一次被外键引用的全表重建**（`0007:6-12` 专门解释了为什么必须 `transaction=off`），且同样要同步 `src/packages/common/field-rules.ts` 的 `TaskType`。
- 若走了这条却没注册 handler，`worker/main.ts:38` 注释写明「未注册的任务类型会被判为 skipped（不静默成功）」——任务永远 `skipped`，**分配永不发生，队列继续积压**，而 `background_tasks` 里一片"正常"。
- 5 分钟固定间隔 + 无 backoff：`0006_rate_limiter` 之外，这条路径对 SQLite 是持续的写放大（每轮 50 INSERT × 288 轮/天），而 `0001` 全库是单文件 better-sqlite3，写锁竞争会直接拖慢用户请求。

**③ 建议**
- 复用现有每日排程的**同一套**幂等范式：`deduplication_key = 'moderate_assign:' + 时间窗`，而不是另起一个裸 `setInterval`。
- 明确失败可见性：分配任务的 `progress` 落 `assigned_count`，并在 `/api/health` 或后台页暴露「待审积压数 / 最近一次分配时间」；否则这个功能死掉了没人知道。
- 给出运维刹车：照 `scheduled.ts:24-44` 的既有做法提供 `DSH_REVIEW_MODERATION_ASSIGN=0` 环境变量硬关。
- 把 `LIMIT 50` 改为**只取未分配过的**（`LEFT JOIN review_assignments ... WHERE ra.id IS NULL`），从根本上消除重复插入。

---

## 五、`reviewers.length === 0` 会让 worker 直接崩（P1）

**① 问题**
`proposal.md:30` 的 `reviewers[i % reviewers.length]`——当库里还没有任何 `role='moderator' AND account_status='active'` 的用户时，`reviewers.length` 为 0，`i % 0` 得到 `NaN`，`reviewers[NaN]` 为 `undefined`，`.id` 抛 `TypeError`。

**② 后果**
- 这是**上线首日必然发生**的路径：迁移刚建完角色，还没人提权，队列就有待审内容。
- 异常若发生在 worker 的 tick 内（`worker/main.ts:104` 的 `onTick`/循环体），轻则该轮心跳/排程中断，重则按 `worker/main.ts:290-296` 的 `process.exitCode = 1` 让**整个 worker 进程退出**并进入重启循环。而 worker 一并负责 `review_check`/`review_publish`/`comment_check`/`comment_publish`——即**审核链路的自动处理全线停摆**，恰好在最需要它的时候。

**③ 建议**
先判空并**带日志地跳过**（不是静默 return）：
```ts
if (reviewers.length === 0) { log.warn('无可分配审核员，跳过本轮', { pending: pending.length }); return; }
```
并把「无 active 审核员」做成健康检查告警项——这是可观测性缺口，不是异常处理细节。

---

## 六、其余 SRE 视角的次要问题

| # | 问题 | 位置 | 后果 | 建议 |
|---|---|---|---|---|
| 1 | 分配过程**不写 `operation_records` 审计** | `proposal.md:26-32` | 与 `0042_audit_immutable.sql:4-11` 确立的「服务层关键路径必须落审计」口径脱节；事后无法回答"这条内容是分给谁、谁分配的" | 分配与改派都落 `operation_records`（`actor_kind='system'`，带 `task_id`，符合 `0001:245-250` 的 CHECK 组合） |
| 2 | 没有分配**回收/超时**机制 | `proposal.md:15-21` | 审核员离职/账号 `disabled`（`account_status`，`0001:30`）后，其名下任务永久滞留，队列"看起来已分配"实则无人处理——积压问题被伪装成已解决 | 加 `status`/`expires_at`，每轮扫回超时未处理者重新分配 |
| 3 | 「轮询」无状态，审核员增减会重排 | `proposal.md:28-32` | 新增一个 moderator 后 `i % length` 全体位移，正在处理的人发现任务易主 | 改为按当前**待办负载最少**分配（`COUNT(未完成)`），或按 id 稳定哈希 |
| 4 | 审核入口沿用 `/admin/submissions` | `proposal.md:36` | 若该路由是 `role='admin'` 硬校验，moderator 会 **403**；方案未核实 | 上线前实测 moderator 账号访问该路由；并给 403 明确的用户可见提示而非空白页 |
| 5 | 迁移未说明**回滚** | `proposal.md:9-21` | 本仓库无 down 迁移机制（`migrate.ts` 只做 up + `schema_migrations` 去重），角色闭集一旦扩到 `moderator` 就**无法撤回** | 在方案里写明"不可回滚"，或选择复用 `user_roles`（第一条建议）从而天然可回滚 |
| 6 | 预期收益不可度量 | `proposal.md:44-46` | 「减少管理员负担」无基线、无指标，上线后无法判断成败，也无法设告警 | 给出可观测指标：待审队列深度、P95 待审时长、审核员人均待办数，并接入现有诊断脚本 |

---

## 结论

**不建议按现方案上线。** 三条**阻断项**：

1. **`ALTER TABLE ... DROP CONSTRAINT` 在 SQLite 不存在**（`proposal.md:9`）——迁移必失败，且因 `instrumentation-node.ts:33-41` 刻意不退出进程，故障会以"服务看着正常"的形态漏到生产。
2. **每 5 分钟扫描器没有落点**（`proposal.md:23`）——扩 `task_type` 需再重建全表，漏注册 handler 则任务全部 `skipped`、分配静默不发生。
3. **`reviewers.length === 0` 的除零**（`proposal.md:30`）——上线首日必触发，可能连带整个 worker 停机，波及所有审核自动处理。

**最省事的正确解**：审核员角色**不要动 `users.role`**，直接复用 `0040_user_roles.sql` 的 `role_type='reviewer'`（`0040:29` 已有该取值）。这样第三条 P0 的"重建 users 表"风险直接归零，`review_assignments` 只需补上 `UNIQUE(submission_id)` 与两个索引，扫描器照 `0007`/`scheduled.ts` 的既有幂等范式接入即可。