# n46-ownership-dispute · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0025
> 模型 deepseek-chat · 7137 字 · 32 次工具调用

---

# SRE 评审：n46 插件归属争议的申诉流程

我把 proposal.md（全文 30 行）与仓库实际约定做了对照：migrations 0028（申诉表先例）、0040（`plugin_ownerships` 实际结构）、0042（审计不可变）、0012（防刷部分唯一索引），以及 `src/server/platform/migrate.ts`、`db.ts`、`tx.ts`、`errors-http.ts`、`role 目录` 的 `ownership.ts`。

**总体判断：这份方案按当前形态上线，会给运维带来三个 P0 级后果：线上启动失败、大规模超时排队、运维无法追查的归属改写。** 数据库结构那一段尤其严重，因为仓库里 `plugin_ownerships` 的结构（0040）和提案假设的并不一样，而提案完全没提这一点。

---

## P0-1 迁移编号 0043 已建 `ownership_disputes`，但字段假设与真实结构不符；按提案 SQL 直接落地，线上启动会报错失败

**① 哪部分有问题**
第 9-19 行的建表 SQL。提案写的是：
- `created_at INTEGER NOT NULL`（无 `CHECK (>=0)`）
- `status TEXT NOT NULL DEFAULT 'pending'`，没有 CHECK、没有处理人、没有处理时间、没有结论字段
- `plugin_id TEXT NOT NULL REFERENCES plugins(id)`，没写 `ON UPDATE/ON DELETE`

**② 后果**
- **外键行为与全站不一致**：`0015:22-23`、`0012:39-40`、`0040:89` 每一处引用都显式写了 `ON UPDATE RESTRICT ON DELETE CASCADE/RESTRICT`。提案漏写，SQLite 会采用默认 `NO ACTION`。后果是：删插件（`DELETE FROM plugins`）时，一旦存在争议行，就会直接 `FOREIGN KEY constraint failed`；而 `0012:39` 的收藏表在 `ON DELETE CASCADE` 下会静默失败。也就是说，**争议表会让“删插件”这个运维操作从“能删”变成“删不掉，还报 500”**（经 `errors-http.ts:63` 兜底折成 `internal`）。
- **`created_at` 无 `CHECK (>=0)`**：`migrate.ts` 的 `REQUIRED_TABLES`（`migrate.ts:141-...`）和全库校验（`db.ts:64` 的 `foreign_keys` 硬断言）都依赖约束一致。漏掉 CHECK 不会立刻出错，但会让“后台填写错误时间戳”这类数据污染通过写入。
- **缺处理人/处理时间/结论，直接导致 P0-3（见下）无法审计。**

**③ 建议**
```sql
id TEXT PRIMARY KEY,
plugin_id TEXT NOT NULL REFERENCES plugins(id) ON UPDATE RESTRICT ON DELETE CASCADE,
claimant_id TEXT NOT NULL REFERENCES users(id) ON UPDATE RESTRICT ON DELETE RESTRICT,
evidence TEXT NOT NULL CHECK (length(trim(evidence)) BETWEEN 20 AND 4000),
state TEXT NOT NULL DEFAULT 'pending' CHECK (state IN ('pending','upheld','overturned','withdrawn')),
-- 快照：申诉那一刻的归属现场（照 0028:25-27 的做法）
snapshot_owner_user_id TEXT, snapshot_ownership_source TEXT, snapshot_ownership_status TEXT,
resolution_note TEXT, resolved_by TEXT REFERENCES users(id), resolved_at INTEGER,
created_at INTEGER NOT NULL CHECK (created_at >= 0),
CHECK ((state='pending') = (resolved_at IS NULL)),
CHECK ((state='pending') = (resolved_by IS NULL))
```
并补两条索引：`UNIQUE(plugin_id) WHERE state='pending'`（防同一插件被反复开单，照 `0028:51-52`）、`(state, created_at, id)`（照 `0028:55`）。

---

## P0-2 「任何用户都可以对任意插件提出争议」 + 无频率限制 = 管理员队列被刷满，且新的申诉流程会写入主库

**① 哪部分有问题**
第 22 行“任何用户都可以对任意插件提出归属争议”。

**② 后果**
- 全站**所有**用户提交入口都在 `rate-limit.ts:57-133` 的 `RateLimitAction` 里登记了额度（`report` 20/小时、`correction` 10/小时、`role_application` 3/天）。这条新的对外提交路径**完全没有额度**，`RateLimitAction` 联合类型里也没有对应成员——运维侧等于开了一个不受约束的写入口。
- 更关键的是：`db.ts:15-17` 明确“better-sqlite3 是同步驱动，等待会阻塞所属线程”，`db.ts:57-60` 只有 WAL + `busy_timeout=5000`。刷入的争议行 + 管理员处理时的 `UPDATE plugin_ownerships` 全在同一个进程、同一条写锁上。**刷得越快，正常用户越容易撞 5000ms busy_timeout 后拿到 SQLITE_BUSY。**
- 队列侧：提案没有任何“同一插件只能有一条待处理争议”的约束，同一插件可以被 N 个账号各开一条，管理员要人工去重。

**③ 建议**
1. 在 `rate-limit.ts:57` 加 `| 'ownership_dispute'`，按天计（参照 `role_application` 3/天，`rate-limit.ts:127-133`）。
2. 表上加 `UNIQUE(plugin_id) WHERE state='pending'`（对应 P0-1 的索引），让“重复开单”由数据库返回 409，不依赖应用层先查后插（`0028:22-24` 就是这么做的，并且明确写了理由）。
3. 申诉人应为“该插件当前归属记录的持异议方”，不能任意插件。服务层至少校验该插件存在且 `plugins.id` 有效，并考虑要求申诉人先有一条 `plugin_ownerships` 的 `rejected`/`pending` 记录——否则任何人对任何插件都能开单，运维无法分辨有效争议。

---

## P0-3 「管理员直接改 `plugin_ownerships` 的归属人」——这句话违反了库里的部分唯一索引和事务纪律，运维会遇到 500 且无审计

**① 哪部分有问题**
第 25 行“直接改 `plugin_ownerships` 的归属人”。

**② 后果（结合真实代码）**
- **会直接违反唯一索引**：`0040:118-120` 有 `uq_plugin_ownerships_verified ON plugin_ownerships(plugin_id) WHERE status='verified'`。所谓“改归属人”，如果实现成把 A 的 `verified` 直接 UPDATE 成 B，或者给 B 插一条 `verified` 而不先把 A 那条降级，就会立即被 `UNIQUE constraint failed` 拦下。`ownership.ts:325-327` 的 `reviewClaim` 里已经踩过并留下注释（“唯一索引会拦住第二个 verified”）。改归属必须是 **“先把旧行降级到 rejected，再走一遍 verify”** 的两步，且必须在**同一个** `withTransaction` 里（`tx.ts:43`），否则中途失败会留下“一个 verified 都没有”或“两个 pending”的中间态。
- **无留痕，运维无法追查**：提案改完归属后只在 `ownership_disputes` 落一行 `status`（连处理人字段都没有）。而全站审计是 `operation_records`（`operations/service.ts:33-44`，`UNIQUE(action_type, operation_key)`），并且 `0042:15-23` 刚加了 `trg_operation_records_no_delete` / `_no_update` 两条触发器**堵死事后篡改**。也就是说：审计表已经被加固成不可改，但**这次归属改写根本不写审计**，等于在加固后的审计墙上开了一个口子。归属权是“开发者身份门槛”的判据（`ownership.ts:8-12`），改错了没有任何回溯依据。
- **通知一定失败**：真实流程在审核完成时会调 `deliverClaimResult`（`ownership.ts:355-365`），去重键里带 `reviewedAt` 轮次（`ownership.ts:169-174` 明确写了“去掉轮次会让第二次通知被 DO NOTHING 吞掉”）。提案没提通知，运维上线后“归属已被改，但原归属人毫不知情”，客服工单会直接回流到 SRE。

**③ 建议**
明确写进方案：
- 改归属 = `reviewClaim` 的复用/扩展，**必须在单事务内**：①旧行 `status='rejected'` 并写 `reviewer_id`/`reviewed_at`（注意 `0040:113` 的 `CHECK (status <> 'pending' OR reviewed_at IS NULL)`——**不要先把旧行改回 `pending`**，会因 `reviewed_at` 非空而违反约束）；②新行置 `verified`；③写 `recordOperationInTx`（`operations/service.ts:129`），`action_type='ownership_dispute_resolve'`，`operation_key` 用 dispute id，含 `target_snapshot`（改前的归属）与 `changes`（改后的归属）；④调 `deliverClaimResult` 通知双方。
- `ownership_disputes` 里存**快照**（`snapshot_owner_user_id` / `snapshot_ownership_source` / `snapshot_ownership_status`），照 `0028:25-27` 的理由：事后看 `plugin_ownerships` 已经被改，回看不清“当时争议的是什么”。

---

## P1-1 `evidence TEXT` 无长度/非空约束，管理员界面渲染与库体积都有风险

**① 哪部分有问题** 第 14 行 `evidence TEXT`。

**② 后果** 全库同类字段都强制长度：`0015:29` `description` `<=2000`、`0028:33` `message BETWEEN 5 AND 2000`、`0040:107` `claimed_note <=500`。`evidence` 完全裸奔——单个用户可写入数 MB 文本，SQLite 页面被撑大、备份/`scripts/backup.mjs` 体积暴涨、后台列表一次拉 N 条直接打爆内存。

**③ 建议** `evidence TEXT NOT NULL CHECK (length(trim(evidence)) BETWEEN 20 AND 4000)`。需要附件就走已有的 `file-store`（`platform/file-store.ts`），表里存引用，不要往这张表塞大文本。

---

## P1-2 `plugins` / `users` 的删除与 `ON DELETE` 语义未定义，备份恢复演练会失败

**① 哪部分有问题** 第 13-14 行的两个 REFERENCES 都没写 `ON DELETE`。

**② 后果** `0042:8-12` 记录过一次真实事故：`restore-backup.mjs --force` 整库覆盖 + 缺审计守卫构成伪造通道。对外键定义含糊的表，`restore-backup.mjs` 的整库替换与 `db.ts:64` 的 `foreign_keys=ON` 硬断言组合起来，会在恢复后 `foreign_key_check` 报不一致（`migrate.ts:124` 会把它写进 `foreignKeyCheck`）。运维做恢复演练时会被这条卡住，且分不清是备份问题还是表定义问题。

**③ 建议** `plugin_id ... ON DELETE CASCADE`（插件没了，争议无意义）、`claimant_id ... ON DELETE RESTRICT`（用户被删前必须处理完争议，与 `0015:23` 一致）。并把新表加进 `migrate.ts` 的 `REQUIRED_TABLES` 清单（`migrate.ts:141` 起）——否则启动自检不会发现这张表建漏了。

---

## P1-3 方案缺少状态机终态与“驳回后能否重开”，运维会收到无解工单

**① 哪部分有问题** 第 25 行“管理员查看证据后，直接改归属人”是全部处理流程描述。

**② 后果** 对比 `0028:29-30` 把状态收敛成三个终态（`pending/upheld/overturned`）并显式说明“不做 partially 这类中间态”。提案只有一个 `pending` 默认值，没定义：
- 管理员判“维持原判”时那行怎么变？
- 申诉人败诉后能不能再申诉（若能，同一插件会无限开单 → 回到 P0-2）？
- 原归属人**在改归属时**是否也应有申辩通道？

这些不是产品问题，是**值班问题**：每一条缺失都会变成“用户来问，运维查不到状态该是什么”的工单。

**③ 建议** 明确定义 `state IN ('pending','upheld','overturned','withdrawn')`，`upheld`/`overturned` 必须有 `resolved_by`+`resolved_at`（用 P0-1 的两条 CHECK 强制），并对“驳回后重新申诉”给出去重口径（建议：同一插件同一申诉人，`upheld` 后 30 天内不许再开，靠索引或服务层实现，并在方案里写清选的是哪种）。

---

## 小结（按上线阻塞程度排序）

| 级别 | 问题 | 一句话后果 |
|---|---|---|
| P0 | 表结构与全库约定不一致（缺 `ON DELETE`/CHECK/终态） | 删插件报 500，迁移与自检口径分裂 |
| P0 | 任意用户任意插件 + 无 rate limit | 经 `rate-limit.ts` 的其它入口口径对比，这是唯一裸奔写入口；WAL 单写锁下正常用户撞 `busy_timeout` |
| P0 | “直接改归属人”绕过 `uq_plugin_ownerships_verified` 与 `operation_records` 审计 | 违反唯一索引 → 500；归属改写零留痕，而审计表刚被 0042 锁死不可改 |
| P1 | `evidence` 无长度约束 / 未登记 `REQUIRED_TABLES` / 状态机未定义 | 库与备份膨胀、自检漏检、值班无标准答案 |

最小可上线版本需要：把 P0-1 的 SQL 改成上面那版、加 `ownership_dispute` 频率限额与 `UNIQUE(plugin_id) WHERE state='pending'`、把“改归属”明确表述为 **“单事务内降级旧行 → 置新行 verified → 写 `recordOperationInTx` → 通知双方”**。这三条不改，我不建议上生产。