# 评审：插件归属争议申诉流程 · 性能工程师视角

> 评审对象：`atb/tasks/n46-ownership-dispute/environment/proposal.md`（30 行）
> 评审人身份：**性能工程师**（只评容量、查询复杂度、索引、锁与写放大、部署与运维成本；不评权限模型、合规、UI）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**不能按现方案上线**。方案本身只有 8 行 SQL + 3 句流程描述，**通篇没有任何一条性能约束**；
> 而它引入的是一条「任何登录用户 × 全站插件」的可写攻击面（§2），和一条**没有索引上限、没有分页**的
> 后台查询路径（§3）。数据库是单文件 SQLite（`better-sqlite3` 单写者），这两件事的代价被放大了：
> 写锁是全库的，读是同步阻塞在 Node 主线程上的。

---

## 摘要（前 5 条最重要）

1. 【严重】**§2「任何用户都可以对任意插件提出归属争议」= 一张可被放大 10⁴ 倍的写放大炸弹，且绕开了本仓既有的频控体系。**
   依据：`proposal.md:23`（无任何前置校验）；`src/app/api/me/plugins/[pluginId]/claim/route.ts:14-32`（本仓现有的认领接口**也没有**频控，可作对照基线）；
   `src/server/platform/rate-limit.ts:68-134`（9 项 `RATE_LIMIT_RULES` 一个都没给归属/争议留位置）；
   `src/app/api/me/roles/applications/route.ts:41-56`（本仓「人工审核队列类」接口的标准写法：`enforceRateLimit` + `refundQuietly`）。
   后果：一次脚本化的 POST 循环即可写入 O(插件数) 行，而 `ownership_disputes` **既没有唯一键也没有索引**（`proposal.md:12-19`），
   管理员页每次打开都要全表扫。插件目录是按总数计费的（`src/server/modules/catalog`），几千行争议就是几千次行插入 + 一次全表扫描，
   在单文件 SQLite 上足以把 web 进程的写锁占到「其他所有写请求排队」的程度。

2. 【严重】新表**零索引**，而它唯一的使用场景就是「按状态筛 + 按时间排」的后台队列。
   依据：`proposal.md:12-19`（只有 PK）；对照本仓同类队列表 `migrations/0040_user_roles.sql:64-70`：
   `role_applications` 建了 `idx_role_applications_user` 与 `idx_role_applications_queue(status, created_at, id)`，
   而 `plugin_ownerships` 更是建了三条（`0040:113-115` + 部分唯一索引 `:119-120`）。
   后果：管理员每点一次「待审」页签就是一次 `WHERE status='pending' ORDER BY created_at` 的全表扫描 + 排序（走临时 B-tree，
   受 `SQLITE_TMPDIR` / 内存限制约束），行数一多就从「几十毫秒」掉到「秒级阻塞 Node 事件循环」。

3. 【高】没有 `UNIQUE(plugin_id, claimant_id)`，争议表会**无界增长**并直接推高上一条的扫描成本。
   依据：`proposal.md:13-18`；对照 `0040_user_roles.sql:69-70`（`uq_role_applications_pending`：同一用户同一身份只允许一条待审）。
   后果：同一个人对同一插件刷 1000 次，就是 1000 行；「管理员先看到最新那条」这件事在无索引时是 `ORDER BY created_at DESC` 全扫。
   建议：照 0040 的写法用**部分唯一索引**（`WHERE status='pending'`），既防重复又不挡「被驳回后再申诉」。

4. 【高】§3「管理员直接改归属」在实现上必然是「旧行 UPDATE 成 rejected + 新行 INSERT/UPDATE 成 verified」——
   这条写路径**没有事务边界与锁序**的描述，且会踩到 `uq_plugin_ownerships_verified` 的部分唯一索引；
   更关键的是它**不写审计**，而审计表是全站**唯一**的追加式写入热点（`0042_audit_immutable.sql:17-27` 已禁止 UPDATE/DELETE）。
   依据：`proposal.md:26`；`migrations/0040_user_roles.sql:119-120`；`src/server/modules/roles/ownership.ts:329-332`（现有的「先查后写」保护）。
   后果：一次归属变更 = 2~3 次写 + 1 次审计 INSERT，全部在**同一个 SQLite 写事务**里；方案没提这些写必须批量/按固定顺序，
   并发两个管理员改同一插件时，后到的会撞唯一索引报错（业务上是 500，见 `ownership.ts:239-250` 注释里已复现过的同类事故）。

5. 【中】方案漏掉了「争议产生 → 审核 → 结果通知」这条链路上**本仓已经存在**的异步/去重负担。
   依据：`proposal.md:28-30` 的「预期」只有两句，没有任何通知与状态推送；
   `src/server/modules/roles/ownership.ts:173-200`（认领结果通知的 `deduplicationKey` 带轮次 `reviewedAt`，否则第二次结论会被 `ON CONFLICT DO NOTHING` 吞掉）；
   `migrations/0041_role_notifications.sql:39`（`notifications.kind` 是硬 CHECK 枚举，新增 kind 必须**重建整张表**，见 `:36-67`）。
   后果：如果按"照抄认领流程"来实现争议结果通知，会引入一个**全表重建迁移**（notifications 已是第 6 次重建，`0034/0036/0037/0041` 都干过），
   每次重建期间 `PRAGMA foreign_keys = OFF`（`0041:32`），对正在跑的 web/worker 是一段**不可并发写入的窗口**。这不是代码问题，是发布期的可用性问题。

---

## 完整问题清单（按严重度）

### P-01 【严重】写放大：任意用户 × 任意插件，无频控、无前置校验
- **现场**：`proposal.md:23`：「任何用户都可以对任意插件提出归属争议，填写证据。」——没有一句提到限流、校验、冷却。
- **仓库现状对照**：
  - 现有认领接口同样**没有**频控：`src/app/api/me/plugins/[pluginId]/claim/route.ts:14-32`（只有 `assertCanWrite`）。
    即便如此，认领天然被 `UNIQUE (plugin_id, user_id)`（`0040_user_roles.sql:111`）挡住，同一个人对同一插件最多一行；
    争议方案**连这个天然上限都没有**（见 P-03），所以不是"和现有一样平"，而是**严格更差**。
  - 同一「人工审核队列」形态的接口是怎么写的：`src/app/api/me/roles/applications/route.ts:41-56`
    ——`enforceRateLimit(db, 'role_application', userId, 1, …)`，并且**失败时 `refundQuietly` 退还额度**（`:35-55` 的注释写明了「用户点错 3 次当天额度就烧光」这个坑）。
- **后果**：
  - 单用户批量脚本 → 每插件一行 → 表行数与插件总数同阶（全站插件目录规模，`catalog` 模块常态在千级）。
  - SQLite 是**单写者**：每行一个 `INSERT` 各自一个事务，等于每秒几十次 `BEGIN IMMEDIATE`/COMMIT，
    其他写请求（评分、评论、通知）会被 `SQLITE_BUSY` 或忙等（`src/server/platform/db.ts` 的 busy_timeout 决定是报错还是拖慢）。
  - 管理员队列页随之线性变慢（见 P-02），"疲于应付"在性能上是**确定的**，不是比喻。
- **建议**：
  1. 给 `rate-limit.ts:46-55` 的 `RateLimitAction` 加 `'ownership_dispute'`，在 `RATE_LIMIT_RULES:68-134` 里照 `report`/`role_application` 的口径定值（例如 5 次/小时/用户），
     并在 `settings/index.ts` 里补 `rate_limits.ownership_disputes_per_hour` 配置项（与 `rate_limits.role_applications_per_day` 同形，`:262-271`）。
  2. 路由层照 `role-applications/route.ts` 的写法：先 `enforceRateLimit`，失败路径 `refundQuietly`——校验没通过不该烧额度。
  3. 批量防护再加一层便宜的：按 `claimant_id` 计「当天未处理争议数」，超过 N 直接拒绝（一条 `COUNT(*)` + 索引即可，见 P-02 的索引）。

### P-02 【严重】新表无任何索引，后台队列必然是全表扫描
- **现场**：`proposal.md:12-19` 只有 `id` 主键。
- **本仓同类表的既定写法**（这是可抄的范本，不是我发明的优化）：
  - `migrations/0040_user_roles.sql:64-65`：
    `idx_role_applications_user(user_id, created_at DESC, id)` + `idx_role_applications_queue(status, created_at, id)`；
  - `migrations/0040_user_roles.sql:113-115`：`plugin_ownerships` 三条索引，其中 `idx_plugin_ownerships_queue(status, created_at, id)` 完全同构；
  - 现有队列页正是按这个索引排序取数的：`src/server/modules/roles/ownership.ts:296-310`（`WHERE o.status = ? ORDER BY o.created_at ASC, o.id`）。
- **后果**：
  - 管理员页每次打开 = 全表扫描 + 排序 + 再 `JOIN plugins / users`（如果页面要显示插件名与申诉人，方案虽然没写，但必然要）；
  - 全表扫描发生在 `better-sqlite3` 的**同步**调用里，直接阻塞 Node 事件循环（全站其他请求一起卡）；
  - 加上 `proposal.md:15` 的 `evidence` 是自由文本（无 `length()` 上限），大行会让扫描的 I/O 成本再放大一截——
    对照 `0040:89` 的 `claimed_note … length(claimed_note) <= 500` 与 `:57` 的 `review_note` 同款约束，本仓对「用户填写文本」一向上限硬约束。
- **建议**：
  ```sql
  CREATE INDEX idx_ownership_disputes_queue ON ownership_disputes(status, created_at, id);
  CREATE INDEX idx_ownership_disputes_plugin ON ownership_disputes(plugin_id);
  CREATE UNIQUE INDEX uq_ownership_disputes_pending
    ON ownership_disputes(plugin_id, claimant_id) WHERE status = 'pending';
  -- 证据文本必须有上限（与 0040:57/89 同口径）
  -- evidence TEXT CHECK (evidence IS NULL OR length(evidence) <= 2000)
  ```

### P-03 【高】无唯一约束 → 表无界增长，且与 P-02 的扫描成本叠加
- **现场**：`proposal.md:13-18` 没有 `UNIQUE(plugin_id, claimant_id)`，也没有「同一插件同时允许多少条待审」的约束。
- **对照**：`0040_user_roles.sql:111` `UNIQUE (plugin_id, user_id)`；`:69-70` `uq_role_applications_pending … WHERE status = 'pending'`。
  本仓的既定口径是：**待审态唯一，已结束态不唯一**（这样「被驳回后能再申诉」与「不能无限刷」同时成立）。
- **后果**：见 P-01 的写放大；同时「管理员先处理哪条」变成 `ORDER BY created_at DESC` 全扫后取第一条，行数越多越慢，
  且无法用 `LIMIT 20` 做分页（方案没提分页，`listQueue` 现有的实现同样**没有分页**：`ownership.ts:296-310` 一次取全量——
  这是现有代码的问题，但新表行数会比认领表大得多，照抄就会踩同一个坑）。
- **建议**：用部分唯一索引（同 P-02）；队列查询必须带 `LIMIT` + 游标（`(created_at, id)` 作为游标键，与索引顺序一致），
  不要引入 `OFFSET`（SQLite 上 `OFFSET n` 仍要扫描前 n 行）。

### P-04 【高】§3「直接改归属」的写事务没有描述，会踩部分唯一索引 + 现有保护
- **现场**：`proposal.md:26`：「管理员查看证据后，直接改 `plugin_ownerships` 的归属人。」
- **仓库约束**：
  - `migrations/0040_user_roles.sql:119-120`：`uq_plugin_ownerships_verified ON plugin_ownerships(plugin_id) WHERE status = 'verified'`
    ——**必须先撤销旧行再建新行**，顺序反了就撞约束；
  - `src/server/modules/roles/ownership.ts:327-337`：现有 `reviewClaim` 的标准写法是「先 `SELECT` 一次拿人话错误 → 再 `UPDATE`」，
    `:239-250` 的注释记录了历史上「没做 taken 检查 → 撞部分唯一索引 → 500」的真实事故；
  - `src/server/modules/roles/ownership.ts:173-200`：审核结果通知在同一事务内投递（`deliverClaimResult`），
    且 `deduplicationKey` 必须带轮次 `reviewedAt`，否则第二次结论会被 `ON CONFLICT DO NOTHING` 吞掉。
- **后果**：
  - 一次归属变更 = `UPDATE 旧行` + `UPDATE/INSERT 新行` + `INSERT operation_records`（`0001_baseline.sql:411-431`，`:426` 有 `UNIQUE(action_type, operation_key)`）
    + `INSERT notifications`，**4 次写**在同一写锁内完成。方案没有说要合成一个事务、也没说要按固定顺序，
    并发两个管理员处理同一插件时会在唯一索引上失败（对用户是 500，因为 `errors-http` 只映射 `DomainError`）。
  - 更实际的性能点：如果实现者偷懒写成「先 DELETE 旧行再 INSERT 新行」，会**毁掉** `plugin_ownerships` 上的三条索引与历史记录，
    同时让 `countVerifiedForUser`（`ownership.ts:369-375`）在事务中间看到 0 —— 开发者门槛判定会瞬时抖动。
- **建议**：
  1. 复用 `reviewClaim` 的事务骨架（`withTransaction` + 先查后写 + 冲突给 `err.conflict`），**不要**新写一套；
  2. 顺序固定为「降级旧行 → 提升/新建新行 → 写审计 → 发通知」，全部在一个 `withTransaction` 内（`src/server/platform/tx.ts:42-52` 明确要求回调同步）；
  3. 查询侧要能扛住 `countVerifiedForUser` 的调用频度：该函数被 `roles/service.ts:472-483` 在「开发者资格」判定中调用，
     只靠 `idx_plugin_ownerships_user(user_id, status, created_at DESC)`（`0040:113`）已可覆盖（前导列是 user_id），**不需要**新增索引——
     但**如果**实现者把「争议审核」做成"每次打开后台都调一次全量资格重算"，就会变成 N 次这类查询，那才需要改。

### P-05 【高】争议处理没有审计 → 事后核查只能全表比对，运维成本无从估算
- **现场**：`proposal.md:26-30`，「预期」里只有「有据可查」「可纠正」，但没有任何一条写「记入 `operation_records`」。
- **仓库约束**：`operation_records` 是唯一审计载体（`0001_baseline.sql:411-431`），
  `0042_audit_immutable.sql:17-27` 已加 `BEFORE UPDATE/DELETE` 的 `RAISE(ABORT)` ——**只允许追加**。
  追加式表的好处正是「写便宜、读慢也不怕」；一旦漏写，事后想重建"谁在何时改了归属"就必须
  把 `plugin_ownerships` 与备份做全库 diff，成本从「一条索引查询」变成「一次全库比对」。
- **建议**：在 §3 里把审计写进去（`recordOperationInTx`，`src/server/modules/operations/service.ts:168-174`），
  且**与业务变更同事务**——这也顺手解决了 P-04 的事务边界问题。

### P-06 【中】通知链路的迁移成本：`notifications.kind` 是硬 CHECK，新增 kind 要整表重建
- **现场**：`proposal.md:28-30` 完全没有提「处理结果如何通知申诉人/被申诉人」。
- **仓库现状**：`migrations/0041_role_notifications.sql:36-67` —— 为了加两个 kind，整张 `notifications` 表被
  `CREATE new → INSERT SELECT → DROP → RENAME` 重建一遍，期间 `PRAGMA foreign_keys = OFF`（`:32`），
  文件头 `:1` 还标了 `-- dsh:migration-transaction=off`。这已经是通知表的**第 6 次**重建（0034 / 0036 / 0037 / 0041 …）。
- **后果（发布期可用性，不是查询性能）**：
  - 重建期间 web/worker 若有并发写入，会因 `foreign_keys=OFF` 而**不受外键保护**；重建耗时与 `notifications` 表行数成正比
    （用户越多越慢），而这一步发生在 `migrate` 容器里，`compose.yaml:81-83` 是 `worker: depends_on: migrate: service_completed_successfully`
    ——**迁移越慢，整个站点就绪越晚**。
  - 如果争议通知复用现有 `plugin_claim_result`（`0041:39`）而不加新 kind，则这次重建可以省掉。
- **建议**：优先复用 `plugin_claim_result`（detail 里写「归属已变更」）以避免再重建一次表；
  若一定要新 kind，把它与其它待发版的新 kind **合并成一次重建**，并在发布说明里标注这段停写窗口。

### P-07 【中】争议→处理→再申诉的**状态流转**没有定义，会导致「同一插件反复进队列」的重复查询
- **现场**：`proposal.md:17` 只有 `status TEXT NOT NULL DEFAULT 'pending'`，取值集合、终态、可否重开都没写。
- **后果**：`status` 无 CHECK（这是正确性问题的表亲），但在性能上表现为：无法用一条 `WHERE status='pending'` 的
  部分索引把「活跃争议」与「历史争议」分开，历史行会和活跃行一起被扫。
  `0040_user_roles.sql:87-88` 的做法是 `status TEXT … CHECK (status IN ('pending','verified','rejected'))`；
  `role_applications` 更是四态 `pending/approved/rejected/withdrawn`（`:52-53`）。
- **建议**：定死 4 个取值（`pending / upheld / rejected / withdrawn`），配合 P-02 的部分唯一索引，
  让「活跃争议」的查询与写入都只碰一个小集合。

### P-08 【中】证据文本与附件没有容量约束
- **现场**：`proposal.md:15` `evidence TEXT` —— 无 `length()` 上限；方案也没说证据是纯文本还是文件。
- **后果**：如果实现成「附件直接塞 BLOB 或 base64 进这一列」，单行可达 MB 级，`ownership_disputes` 的**全表扫描成本与表大小无关地暴涨**
  （SQLite 溢出页），且 `backup` 周期任务（`deploy/dsh-review-backup.service/timer`）的备份体积同步膨胀。
  本仓对用户文本的一贯做法是硬上限：`0040:57`（`review_note` ≤ 500）、`:89`（`claimed_note` ≤ 500）。
- **建议**：`evidence TEXT CHECK (evidence IS NULL OR length(evidence) <= 2000)`；附件走已有的媒体/附件链路（参照 `feedback_attachment_links`），
  不要往这张"高频扫描表"里塞大对象。

### P-09 【中】新表没有纳入自检与诊断清单 → 静默缺表不会被发现
- **现场**：`proposal.md` 未提及运维登记。
- **仓库约束**：`src/server/platform/migrate.ts:142-237` 的 `REQUIRED_TABLES`（0040 的 `plugin_ownerships` 就在 `:236`），
  自检 `selfCheck`（`:239-267`）据此判定 `/api/health`，`src/server/platform/health.ts:61-63` 复用它。
- **后果**：没有登记的话，表被误 DROP 后 `/api/health` 仍 200，争议页静默 500——与 `migrate.ts:214-215` 里记录过的
  「讨论区 10 张表曾漏登记，自检报缺表却什么都不缺，等于把这道门禁废掉了」是同一类事故。
- **建议**：在 `REQUIRED_TABLES` 追加 `'ownership_disputes'`，并在 `scripts/diagnose.mjs` 的清单里同步（该脚本的清单已与 `migrate.ts` 漂移，见 n44 评审 S-07）。

### P-10 【中】容量与备份：单文件库 + 只增不删的表，没有归档策略
- **现场**：`proposal.md` 通篇无保留期（retention）设计，`status` 也没有终态清理路径。
- **后果**：争议行永久保留 + `plugin_ownerships` 的归属变更历史无法在业务表里看出（P-05 的审计是唯一来源），
  意味着 `backups/` 里必须长期保留 `operation_records`，且 `ownership_disputes` 只增不减。
  在 SQLite 单文件上，这不是"磁盘涨一点"，而是 **P-02 的全表扫描随时间单调变慢**，没有自然回落的时刻。
- **建议**：明确「已结束争议保留 N 天后归档/删除」——注意 `operation_records` 是**不可删**的（`0042`），
  所以归档只能对 `ownership_disputes` 本身做，且删除前必须确认审计已覆盖（P-05 先落地）。

### P-11 【低】与既有「认领」流程的查询重叠：两个队列页会问同一张 `plugin_ownerships`
- **现场**：方案新增一条争议队列，但没有说明它与既有 `/admin/plugins/ownerships`（`src/app/admin/plugins/ownerships/page.tsx:47-48`）
  是否复用同一个查询。
- **后果**：若各自实现一份 `JOIN plugins JOIN repositories JOIN users` 的查询，管理员在两页间切换就是两套全表扫描；
  现有 `listQueue`（`ownership.ts:296-310`）已经**没有分页与 LIMIT**，再加一个同构查询只会让后台更慢。
- **建议**：争议队列复用 `MY_PLUGIN_SELECT`（`ownership.ts:79-89`）的 JOIN 形态并加 `LIMIT`，
  或直接在 `plugin_ownerships` 上扩充而不是另开一张表——见结论里的"方案 B"。

### P-12 【低】「现状」一节的说法没有容量含义，但会影响结论口径
- **现场**：`proposal.md:4`：「插件归属（`plugin_ownerships`）目前由系统自动匹配 GitHub 账号或人工认领确定。」
  —— 与仓库一致（`0040_user_roles.sql:76-81`、`ownership.ts:9-18`），**这一句是对的**。
- **性能视角的补充**：正因为 `github_match` 会自动置 `verified`，`plugin_ownerships` 的**已归属行**会随 OAuth 绑定事件自然增长；
  争议流程若每次都重算「这个插件现在归谁」，应走已有索引 `uq_plugin_ownerships_verified`（`0040:119-120`，前导列 `plugin_id`）
  而不是全表扫。方案没写查询形态，建议在 §3 明确「按 plugin_id 查当前 verified 行」，这是 O(log n) 与 O(n) 的差别。

---

## 结论与给方案方的两条落地建议

**现方案在性能上不可放行**：它把「一个人对一台机器说一句我不服」做成了「任何人对全站插件各说一句」，
而承载它的表没有索引、没有唯一约束、没有频控、没有上限、没有分页、没有保留期，落在一个**单写者**的 SQLite 上。

**方案 A（最小改造，保留新表）**——把上面 P-01~P-03、P-08、P-09 一次补齐即可接受：
```sql
CREATE TABLE ownership_disputes (
  id          TEXT PRIMARY KEY,
  plugin_id   TEXT NOT NULL REFERENCES plugins(id),
  claimant_id TEXT NOT NULL REFERENCES users(id),
  evidence    TEXT CHECK (evidence IS NULL OR length(evidence) <= 2000),
  status      TEXT NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending','upheld','rejected','withdrawn')),
  created_at  INTEGER NOT NULL CHECK (created_at >= 0),
  UNIQUE (plugin_id, claimant_id)        -- 同一人对同一插件只有一条（照 0040:111）
);
CREATE INDEX idx_ownership_disputes_queue ON ownership_disputes(status, created_at, id);
CREATE INDEX idx_ownership_disputes_plugin ON ownership_disputes(plugin_id);
```
+ 频控（`RATE_LIMIT_RULES` 新增 `ownership_dispute`）+ 队列 `LIMIT` 游标分页 + `REQUIRED_TABLES` 登记 + 保留期。

**方案 B（更省，值得先评估）**——不新建表：
争议本质是「对一条 `plugin_ownerships` 的 `claimed` 记录提出异议」。
复用现有表意味着**零新索引、零新写路径**：把 `status` 扩成 `pending/verified/rejected/disputed`，
排队查询直接落在已存在的 `idx_plugin_ownerships_queue(status, created_at, id)`（`0040:115`）上；
改变更走现有 `reviewClaim` 的事务骨架（P-04）与审计（P-05）。
代价是 `status` 的 CHECK 变更需要一次表重建（照 `0041` 的手法，`PRAGMA foreign_keys=OFF` 窗口同样存在，见 P-06）。
**性能上方案 B 严格优于 A**：没有第二张需要长期维护扫描路径的表。

---

> 评审人声明：本报告只覆盖性能/容量/部署成本视角。安全（越权、伪造证据）、合规（知识产权申诉期）、
> 业务（谁该被判归属）由其他视角的评审负责，本文不代替它们的结论。
