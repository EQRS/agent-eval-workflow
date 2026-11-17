# 评审：审计记录查询优化方案 · 安全工程师视角

> 评审对象：`tasks/n44-audit-query/environment/proposal.md`
> 评审人身份：安全工程师（只做安全/合规/完整性视角，不评 UI 与性能调优细节）
> 仓库根：`dsh-plugin-reviews-new`（代码在 `src/`、`migrations/`、`docker/`、`deploy/`）
> 结论：**不可放行**。方案新增的 `audit_stats` 是一条**绕过审计不可变保护的可写通道**，且 DDL 本身在 SQLite 中无法执行。

---

## 摘要（前 5 条最重要）

1. 【严重/Critical】`audit_stats` 是可写的"审计视图"，直接架空了 0042 的不可变保护 — 依据：`migrations/0042_audit_immutable.sql:17-27`（operation_records 已加 BEFORE UPDATE/DELETE 守卫）、`proposal.md:11-16`（新表无任何触发器）、`proposal.md:28-32`（页面改读该表）。攻击者只要能用被 DELETE 触发器绕开的 `INSERT ... ON CONFLICT DO UPDATE` 改一行，就能让审计页显示"今天没有敏感操作"，而真实审计行被淹没 — 这正是 0042 立规要堵的"让审计看起来是什么样"的伪造通道。建议：要么给 `audit_stats` 加同款不可变触发器并改由**追加式**统计（只许 INSERT 聚合行，禁 UPDATE/DELETE，以"最后一条生效"读），要么**取消统计表**改为对 `operation_records` 建覆盖索引；任何"页面读什么东西"的决定，其可信度不得超过被读对象的可信度。

2. 【严重/Critical】DDL 自相矛盾，迁移执行即 500：`day TEXT PRIMARY KEY` + `action TEXT NOT NULL`，但写入走 `ON CONFLICT(day, action)` — 依据：`proposal.md:11-15` 与 `proposal.md:23-25`。SQLite 的 ON CONFLICT 目标必须匹配唯一索引/主键，不存在 `(day, action)` 唯一约束会直接报 `ON CONFLICT clause does not match any PRIMARY KEY or UNIQUE constraint`。安全后果不只是"功能不通"：它证明方案**没有在真实 schema 上验证过**，而这份统计将作为"审计呈现"的依据 — 未验证的统计口径不得进入审计链路。建议：PRIMARY KEY (day, action) 或建 UNIQUE(day, action)。

3. 【严重/High】统计与审计不同事务，产生"可被单独操纵的第二事实" — 依据：`proposal.md:21-26` 两条独立语句；`src/server/platform/tx.ts:42-52`、`src/server/modules/operations/service.ts:168-174`（`recordOperationInTx` 明确要求"审计必须与业务变更同事务"）。安全视角看，后者比前者更致命：**统计表能在审计真值之外独立漂移**，事后问责时"页面说有 3 次、库里记了 7 次"无法判断哪边被动过。建议：统计写入必须与 `operation_records` 同一条语句或同一事务（这里建议改为**不落表**：见第 5 条）。

4. 【高/High】审计唯一性语义被破坏：重放会让统计虚高 — 依据：`migrations/0001_baseline.sql:426` `UNIQUE(action_type, operation_key)`；`service.ts:97-134` 同键同摘要时 `replayed: true` **不写新行**；`proposal.md:21-26` 无条件 `count + 1`。更实际的风险在 `src/server/modules/identity/admin.ts:361-374`：幂等回执命中时直接 return，**根本不进审计写入**，所以统计与真值在两个方向上都会偏。建议：统计口径必须与"实际插入了新行"绑定（`replayed === false` 才计），并补一条断言测试（同 operation_key 重放两次后统计只 +1）。

5. 【高/High】方案把"审计页需要什么"和"审计真实性"耦合在一张派生表上，且回答错了问题 — 依据：`src/app/admin/operations/page.tsx:118-132` 真实慢点在两处 **无 LIMIT 的全表 GROUP BY**（actor 维度按 `actor_user_id` 分组、action 维度按 `action_type` 分组），而 `operation_records` 仅有 `idx_operation_target`/`idx_operation_actor`（`0001_baseline.sql:432-433`），没有 `(action_type)` 前导索引 — 所以最便宜且零新写路径的修法是**加索引 + 把分组结果计数缓存**；引入一张新可写表去解决"读慢"，等于为了性能在审计面新开一个写入口，方向性错误。

---

## 详细问题清单（按严重度）

### S-01 【严重】新表无不可变保护，形成"选择性伪造"通道
- 现场：`proposal.md:11-16` 只建表，无 `BEFORE UPDATE/DELETE` 触发器；对照 `0042_audit_immutable.sql:17-27` 与 `0002_triggers.sql:200-223`，本仓对"证据类"表一贯加 RAISE(ABORT) 守卫，全仓已有 35 个触发器。
- 攻击路径（不需要代码权限，只需要一次 SQL）：`UPDATE audit_stats SET count = 0 WHERE day = date('now')` → 管理页"动作分布"直接归零，真实审计行还在但没有人在页面上下钻。
- 这正是 `0042` 文件头注释自述要防的模式（`0042_audit_immutable.sql:11-12`："构成不需要改代码的伪造通道"）。方案等于把刚堵上的口子换了个位置重开。
- 建议：不改统计口径的前提下，最低要求是给该表加与 0042 等价的 UPDATE/DELETE 守卫；但见 S-04，更好的选择是**不建这张表**。

### S-02 【严重】DDL 与写入语句不匹配，迁移即失败
- 依据：`proposal.md:11-15` vs `:23-25`。
- 补充风险：0043 这个编号下如果 `CREATE TABLE` 报错，迁移器按文件名记录，容易留下"文件已登记、表未建成"的半态（`src/server/platform/migrate.ts` 的按文件名去重语义），随后每次发版都会跳过它。
- 建议：`PRIMARY KEY (day, action)`；并在迁移里显式声明"本迁移纯增量、可重放"（与 0042 的写法对齐）。

### S-03 【高】统计/审计双写不一致，且失败方向无定义
- 依据：`proposal.md:21-26`。第二条语句失败时审计已记、统计没记；第一条已写入后抛错回滚时，第二条已执行的 +1 也不会被撤销（两条独立语句意味着**没有共同的事务边界**）。
- 依据：项目纪律 `tx.ts:1-9`「不得各自提交独立事务后假装整体原子成功」。
- 建议：若一定要落表，必须在 `recordWithin`（`service.ts:136-158`）同一条事务里、且在同一 `handle` 上执行；并在事务失败路径上能回滚统计。

### S-04 【高】"派生统计"被升格为审计权威视图是架构性错误
- 依据：`proposal.md:28-32` 页面整块改读统计表，而 `page.tsx:128-132` 现有两块筛选（动作/操作人）依赖 `operation_records` 的真实分布。
- 派生物只能证伪不能证实：统计表一旦与真值背离，安全侧没有任何自动告警（本仓 `/api/health` 只校验表存在，`migrate.ts:142+`，不校验数据一致性）。
- 建议（安全侧最简正解）：**不建统计表**。给 `operation_records(action_type)` 加索引，把 `:118-132` 的两句改成 `SELECT ... GROUP BY ... ORDER BY c DESC LIMIT 50` 并加短 TTL 缓存；`:150-152` 的 `COUNT(*)` 用同法。这样读侧不再 O(N)，审计真值仍是**唯一**事实来源。

### S-05 【高】幂等重放与"无变更"路径使统计系统性偏离
- 依据：`service.ts:115-134`（同键重放直接返回，不写行）、`identity/admin.ts:388-390`（目标已是该角色 → `changed:false`，不写审计）；而方案按"每次写审计记录时"累加（`proposal.md:18-19`）。
- 影响：一条被反复重放的请求会让"看起来发生过 N 次"，与 `UNIQUE(action_type, operation_key)` 保证的"只发生过一次"冲突；统计一旦被用于安全复盘（"某人今天改了几次权限"），会给出错误事实。
- 建议：统计只认"实际插入"，或干脆不统计写侧（见 S-04）。

### S-06 【中】遗漏了两条内联审计写入路径，统计会静默漏计
- 依据：`src/server/modules/identity/admin.ts:276-299`（`insertAudit` 直接 `INSERT INTO operation_records`）、`src/server/modules/catalog/service.ts:545`（同样内联）。全仓共 24 处审计写入调用点，只有 `service.ts` 一个中心入口。
- 安全后果：管理员改角色这类**最敏感**的操作恰恰走内联路径，若统计只在 `recordOperationInTx` 加钩子，结果是"敏感操作不计入统计"——统计失真方向正好偏向掩盖高危行为。
- 建议：若保留统计，必须在**表级**加 `AFTER INSERT ON operation_records` 触发器（而不是在 TS 各调用点补），这样任何写入路径（含未来新增）都被覆盖；触发器方案同时天然消解 S-03 的事务问题（同一语句内完成）。

### S-07 【中】新表未纳入健康检查与诊断清单
- 依据：`src/server/platform/migrate.ts:142-164`（`REQUIRED_TABLES`，现有到 `runtime_heartbeats`）、`scripts/diagnose.mjs:28-34`（另一份手抄清单，已经和 migrate.ts 漂移——少了 0006 之后的若干表）。
- 后果：`audit_stats` 被谁 DROP 后 `/api/health` 仍 200，审计页静默变空/报错，属于"证据资产的静默消失"。
- 建议：两处同步登记；更建议直接引用同一份常量，消掉 `diagnose.mjs` 的重复清单。

### S-08 【中】无保留/清理语义，统计表与审计表生命周期错配
- 依据：全仓无 retention/DELETE 策略；`0042` 会让 `operation_records` 只增不减。方案新增的按天统计表同样是只增。一旦日后为审计表引入保留期清理，统计表的累计值将无法回溯校准（丢弃了明细就无法重算）。
- 建议：任何审计侧的派生数都要写清"可由明细完全重算"这一性质，并在文档里写明重算脚本的存在；否则不要在审计面引入不可重算的派生数据。

### S-09 【中】统计表不含任何篡改取证字段
- 依据：`proposal.md:11-16` 只有 `day/action/count`，无 `updated_at`、无来源水位（如已计入的最大 `operation_records.id`）、无 `request_reference`。
- 后果：连"这张表最后被谁、在什么时候改过"都查不到，出事后无法证明统计曾被人动过（对照 `0001_baseline.sql:411-431` 的完整取证字段设计）。
- 建议：若坚持落表，至少加 `watermark_id`（可核对明细行数）+ `updated_at`，并让页面显示水位与明细计数是否一致。

### S-10 【低-中】`day` 口径未定义，跨时区会对不上 "今天"
- 依据：`proposal.md:25` 用 `today` 变量，但全文未定义其来源（本地时区？UTC？`nowMs()`？）。全仓时间统一用 epoch ms（`0001_baseline.sql:425 created_at INTEGER CHECK (created_at >= 0)`），而统计表用 TEXT 日期，两套时间表示并存。
- 后果：安全复盘"某日发生了什么"时，统计口径与明细口径可能相差一天，属于取证歧义。
- 建议：统一为 `date(created_at/1000,'unixepoch','localtime')` 或 UTC 之一，并写进文档；或改为 `day_ms INTEGER`。

### S-11 【低】权限面：新表给管理页新增一个可写依赖
- 依据：`page.tsx:106-107` 只有 `isAdmin` 守卫；统计表若被其它后台功能顺手写入，会扩大"谁能让审计看起来不同"的主体集合。
- 建议：写入权限收敛到单一入口（触发器或唯一 service 函数），不让任意模块持有 `audit_stats` 的写语句。

---

## 给方案作者的修改建议（安全侧最小集）

1. **首选：删掉 `audit_stats`**。改为 ①给 `operation_records` 加 `(action_type)`（以及必要时 `(actor_user_id, action_type)`）索引；②把 `page.tsx:118-132` 的两处无 LIMIT 全表 GROUP BY 收口加 `LIMIT` + 进程内 TTL 缓存。零新写路径、零新增伪造面。
2. **次选（必须落表时）**：用 `AFTER INSERT ON operation_records` 触发器维护统计（覆盖全部 24 处写路径、天然同事务），`PRIMARY KEY (day, action)`，加 `BEFORE UPDATE/DELETE` 守卫，页面同时展示"统计水位 vs 明细计数"用于对账，并登记 `REQUIRED_TABLES` + `diagnose.mjs`。
3. 无论哪种，补两条测试：①重放同一 `operation_key` 两次，统计只 +1；②直接 `DELETE/UPDATE` 统计表必须 ABORT。

---

## 前提核实（方案对现状的描述是否属实）

| 方案声称（`## 现状`） | 核实 | 结论 |
|---|---|---|
| 后台审计页 `/admin/operations` 查得慢 | `src/app/admin/operations/page.tsx` 存在，`:128-132` 与 `:118-126` 有两个无 LIMIT 全表 GROUP BY，`:150-152` 每次 `COUNT(*)` | **属实**，且慢点定位正确 |
| 优化手段 = 物化统计表 | 仓库中 `audit_stats` 仅出现在方案与本任务文件内（全仓 grep 无 src/migrations 命中） | 属"方案声明新建"，前提合法 |
| `recordOperationInTx(db, input)` 接口存在 | `src/server/modules/operations/service.ts:168-174` | 属实 |
| 「每次写审计记录时」可一处挂钩 | 实为 24 处调用点 + 2 处内联 INSERT（`admin.ts:292`、`catalog/service.ts:545`） | **不实/不完整**，见 S-06 |
| 方案里的 `action` 字段 | `operation_records` 实为 `action_type`（`0001_baseline.sql:413`） | 命名混用，易在实现时挂错列 |

---

## 验证方式（本次评审的证据口径）

- 每条结论均给出"文件:行号"，均经实际读取：`0042_audit_immutable.sql`、`0001_baseline.sql:411-433`、`0002_triggers.sql`（触发器清单）、`operations/service.ts:80-190`、`tx.ts`、`db.ts:30-89`（WAL + `synchronous=FULL`，`:52-55`）、`admin/operations/page.tsx` 全文、`identity/admin.ts:276-299,332-454`、`migrate.ts:141-164`、`scripts/diagnose.mjs:27-34`。
- 未修改任何仓库文件，只读取证。
- S-02 可通过 `sqlite3` 直接复现：按 `proposal.md:11-16` 建表后执行 `:24` 的 upsert，预期 "ON CONFLICT clause does not match any PRIMARY KEY or UNIQUE constraint"。
- S-01/S-06 可复现：建表后执行 `UPDATE audit_stats SET count=0` 不报错即证实无守卫。

## 不适用条件

- 不适用于评审对象不涉及审计/取证类数据的纯读展示改造（例如只改分页 UI）。
- 不适用于方案已明确采用"统计表只追加 + DB 级不可变 + 可重算"三件套并已登记健康检查表清单的情形（此时仅需复核触发器与对账断言）。
- 性能数字（5 秒 → 毫秒）不是本评审的判定依据，安全侧不为其背书。
