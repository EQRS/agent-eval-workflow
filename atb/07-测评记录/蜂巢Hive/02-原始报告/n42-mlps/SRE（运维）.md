# n42-mlps · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0013
> 模型 deepseek-chat · 7292 字 · 21 次工具调用

---

我已核实了足够的证据。以下是 SRE 评审。

---

# SRE 评审：等保二级合规改造方案（`n42-mlps`）

## 结论：**不通过**。方案违反现有架构的 4 条硬性约束，其中 2 条会在上线后造成不可逆的运维事故。且 `0042` 已在 4 个迁移编号前解决了同一问题。

前提说明：我实际读了 `proposal.md`（全文 37 行）、`0042_audit_immutable.sql`、`0040_user_roles.sql`、`0001_baseline.sql`§3.7、`0002_triggers.sql`、`migrate.ts`、`client-ip.mjs`、`tx.ts`、`operations/service.ts`。下面每条都指出文件与行号。

---

## 一、致命问题：新建 `audit_log` 表是重复造轮子，且绕过了已有的防篡改实现

**① 问题所在**：`proposal.md:11-19` 新建 `audit_log` 表。但仓库**已经有**权威审计表，并且刚做过不可变加固：

- `migrations/0001_baseline.sql:411-437` — `operation_records` 表，字段远多于方案：`action_type` / `operation_key` / `actor_kind` / `actor_user_id` / `target_type` / `target_id` / `target_snapshot`(JSON) / `changes`(JSON) / `reason` / `request_reference` / `task_id`，外加 `UNIQUE(action_type, operation_key)` 和两个查询索引。
- `migrations/0042_audit_immutable.sql:20-27` — 已加 `trg_operation_records_no_delete` / `trg_operation_records_no_update`，**审计行不可删不可改**。
- 方案里的 `audit_log` 只有 `id/user_id/action/ip/created_at`，**没有任何不可变触发器**。

**② 后果**：`0043` 会让全站出现**两套审计**，互相不引用。更严重的是方案的 `audit_log` 可被 `DELETE`/`UPDATE`——这正是 `0042` 文件头第 3-13 行记录的已修复安全漏洞（"删除审计行即可让已消费的 operation_key 重新可用……构成不需要改代码的伪造通道"）。新表等于**重新打开同一条伪造通道**，而且等保检查时审计记录可被无声删除，该项直接不达标。运维侧还多一份要备份、要对账、要写保留期清理的数据源。

**③ 建议**：**废弃 `0043_audit_log.sql`**，审计一律写入既有 `operation_records`。若确实缺 `ip` 字段（这是方案里唯一真正的新信息），单独出一个只加列的迁移：
```sql
-- 0043_operation_records_ip.sql
ALTER TABLE operation_records ADD COLUMN client_ip TEXT;
```
注意 `operation_records` 上已有 `BEFORE UPDATE` 触发器（`0042:24-27`），**`ALTER TABLE ADD COLUMN` 不触发 UPDATE 触发器**，所以加列安全；但历史行 `client_ip` 为 `NULL`，需在迁移注释里写明"2026-XX 之前的历史审计无 IP，不可回填"——这点不要试图用 `UPDATE` 补，会被 `0042` 的触发器直接 ABORT。

---

## 二、致命问题：`db.prepare(...).run(...)` 裸写 SQLite，绕过事务封装

**① 问题所在**：`proposal.md:23-29` 的记录点直接调用 `db.prepare("INSERT ...").run(...)`。而本项目对写操作有强制封装：

- `src/server/platform/tx.ts:7-11` 明确规定："跨模块流程创建**一次短事务**，参与模块共用同一受限事务上下文；**不得各自提交独立事务后假装整体原子成功**"。
- `tx.ts:47-56` 提供 `withTransaction`，且 `Tx` 上下文有 `finished` 守卫（`tx.ts:22-38`），事务结束后再用会抛错。
- 已有正确入口：`operations/service.ts:147-156` 的 `recordOperationInTx`，其注释第 149-155 行专门说明"审计必须与业务变更同事务（架构 §6.4）……比让每个模块各自内联一份 INSERT 更能保证不变量只写一遍"。

**② 后果**：`proposal.md:23-29` 的内联 INSERT 会**在业务事务之外**执行。两种失败模式：
1. 业务回滚但审计已落库 → 审计表里存在**从未发生的操作**，等保"有据可查"变成"有据可查但内容是假的"，比没有审计更糟（审计不可信会直接被判不符合）。
2. 业务提交成功但审计 INSERT 失败（磁盘满 / `SQLITE_BUSY`）→ **操作无审计**，且木有异常上报路径。

另外 `clientIp` 在三处反向代理/直连部署下取法不同（见 `client-ip.mjs:56-70`），内联写法必然各处复制粘贴、取法不一致。

**③ 建议**：删除第 23-29 行的实现，改为在**每个写操作的既有事务内**调用 `recordOperationInTx(tx.db, {...})`（`operations/service.ts:147`）。`client_ip` 通过 `resolveClientIp()`（`client-ip.mjs:56`）在 HTTP 入口解析后**作为参数透传进事务**——绝不能在事务内读 header，因为 `tx.ts:9-10` 要求"外部拉取一律在事务之外完成"。

---

## 三、高：`CREATE TABLE audit_log` 没有幂等保护，会打断迁移器与启动自检

**① 问题所在**：`proposal.md:12` 用裸 `CREATE TABLE`（无 `IF NOT EXISTS`）。迁移器 `migrate.ts:37-41` 靠 `schema_migrations` 表按**文件名**去重（`migrate.ts:96-99`），`0042` 文件头第 27 行也确认"迁移器按文件名去重"。

**② 后果**：
- 若曾有人在测试库手工建过 `audit_log`（或某分支已跑过 `0043`），正式库再跑会 `table audit_log already exists`，整个迁移文件回滚，**新版本起不来**。`migrate.ts:104` 起的外键检查 `/ 缺表` 自检会连带失败。
- 方案未把 `audit_log` 登记进 `REQUIRED_TABLES`（`migrate.ts:142-217`）。该清单在 `0037` 处有过教训，注释明确写着"这 10 张此前没登记，自检报'缺表'却什么都不缺，等于把这道门禁废掉了"。新表不进清单，则"表丢了"不会被发现。

**③ 建议**：若仍要建表，必须 `CREATE TABLE IF NOT EXISTS`，并同步在 `migrate.ts` 的 `REQUIRED_TABLES` 追加登记。但按第一条建议废弃该表后，此问题自然消失。

---

## 四、高：审计保留 6 个月 vs. 表已被设为不可删除 —— 方案自相矛盾，运维无法执行

**① 问题所在**：`proposal.md:35` 写"保留 6 个月"。但审计表在 `0042_audit_immutable.sql:20-22` 有 `BEFORE DELETE ... RAISE(ABORT)`。方案没有说明 6 个月后用什么手段清理。

**② 后果**：运维会在 6 个月后执行 `DELETE FROM audit_log WHERE created_at < ?`，然后**直接被触发器 ABORT**（若按建议复用 `operation_records`，则撞 `0042` 的守卫）。清理任务每夜失败报警，运维只能手动 `DROP TRIGGER` —— 破坏了全部不可变性保证，且此操作本身**不留审计**（元问题：审计系统的管理动作无人审计）。同时 `proposal.md:37` 宣称"有据可查"与 6 个月删除承诺在合规口径上冲突，等保二级对审计记录留存通常要求 ≥6 个月且**不得随意删除**，方案需要给出法律/合规依据，否则保留期应由业务方书面裁定。

**③ 建议**：明确保留策略的**执行主体**，二选一并在迁移注释中固化：
- 合规优先（推荐）：审计**永不删除**，6 个月只是"在线可查"窗口；归档走 `VACUUM INTO` 或冷备，不删行。
- 必须删：则不要用 `DELETE`，改为按年**分表 + DROP TABLE**，并在运维手册写明每年执行 `DROP TRIGGER` → expiring → `DROP TABLE` 的有序流程与审批人。

另外，本方案未提**备份与恢复演练**这一运维必备项：`0042` 文件头第 11 行点名 `restore-backup.mjs --force` 整库覆盖是攻击面之一。恢复演练必须验证"恢复后审计触发器依然存在"，建议在发布检查单中加一条 `SELECT name FROM sqlite_master WHERE type='trigger' AND name LIKE 'trg_operation_records%'` 应返回 2 行。

---

## 五、中：`ip` 字段类型与来源未定义，跨反代部署会记录错值

**① 问题所在**：`proposal.md:16` 只有 `ip TEXT`，无 `CHECK`、无长度约束、未说明取值来源。而 `client-ip.mjs:1-12` 明确警告：只有 `DSH_REVIEW_TRUST_PROXY=1` 时才可信 `X-Forwarded-For`，否则"等于让客户端自己决定日志里记的是谁"。

**② 后果**：若部署侧漏配 `DSH_REVIEW_TRUST_PROXY=1`，所有审计 IP 都会是同一条 nginx 地址（`client-ip.mjs:67` 注释已预判此现象）；反之若误开且反代未清洗该头，**攻击者可任意伪造审计来源 IP**，等保的"访问来源可追溯"失效。审计里出现伪造来源比空值更难排查。

**③ 建议**：在迁移中加约束，并把"是否可信代理"记为事实而非隐式配置：
```sql
client_ip TEXT CHECK (client_ip IS NULL OR length(client_ip) <= 45),
client_ip_source TEXT CHECK (client_ip_source IN ('socket','x-real-ip','x-forwarded-for'))
```
把 `client_ip_source` 一并落库，运维事后就能立刻区分"这是真实来源"还是"这是客户端自称"。同时在部署检查单里把 `DSH_REVIEW_TRUST_PROXY` 列为**必检项**，并在启动自检 `migrate.ts` 的 `SelfCheckResult` 里增加该配置的回显（当前 `SelfCheckResult` 第 132-140 行没有它）。

---

## 六、中："所有写操作都记一条"缺乏容量与性能评估

**① 问题所在**：`proposal.md:21` 一句"所有写操作都记一条"，没有任何采样、批量或写入预算说明。`audit_log` 方案连索引都没有（`proposal.md:12-19` 只有主键），而现有 `operation_records` 带 2 个索引（`0001_baseline.sql:436-437`）。

**② 后果**：`audit_log` 无二级索引，6 个月后单表可能达千万行，等保检查时按 `user_id`/时间范围查审计将**全表扫描**，把 SQLite（单写者，`migrate.ts` 自检里 `journal_mode` 可见为 WAL）的读锁拖长，直接拖慢业务写入。运维无法据此出审计报告。

**③ 建议**：按第一条复用 `operation_records`（索引已就绪）即可规避。若坚持新表，必须至少建 `(created_at, id)` 与 `(user_id, created_at, id)` 两个索引，并在方案中给出预估日增量行数与磁盘预算（便于运维规划备份窗口）。

---

## 七、中：二次密码确认未定义失败处理与锁定策略

**① 问题所在**：`proposal.md:31-32`"管理员操作需要二次密码确认"，一句话。未提失败计数、锁定、审计这**二次确认本身**是否记录。

**② 后果**：没有失败限制即可被在线爆破管理员口令，而该路径若失败不写审计（方案未列），事后**无据可查**——恰好推翻自己第 37 行的"有据可查"。运维也无法从日志判断是否正在被攻击。注意现有 `rate-limit.ts` 存在但方案未引用。

**③ 建议**：二次确认接入既有 `platform/rate-limit.ts`，明确"连续 N 次失败锁定 M 分钟"的具体数值，并**将成功与失败都写入审计**（`action_type` 区分 `admin_reauth_success` / `admin_reauth_failure`）。失败记录必须落库，这是等保里最有价值的一类审计。

---

## 八、低：方案未提发布/回滚步骤，不满足运维可执行性

**① 问题所在**：`proposal.md` 全文 37 行，无发布步骤、无回滚方案、无迁移耗时评估。

**② 后果**：`0043` 若含建表 + 索引，在已有数据量的库上需评估锁表时长；本方案的 `CREATE TABLE` 本身很快，但若按建议改为 `ALTER TABLE operation_records ADD COLUMN`，在 SQLite 上是 O(1) 元数据操作，安全。运维若不清楚这点会误判需要停机窗口。

**③ 建议**：方案补两小节：**发布**（`migrate.ts` 的 `dsh:migration-transaction=off` 是否需要——本迁移不需要，参照 `0042:20` 的判断方式）；**回滚**（SQLite 无法 `DROP COLUMN`，加列后回滚需重建表，因此**必须前向修复、不可回滚**，发布前须留存 `VACUUM INTO` 快照）。

---

## 汇总

| # | 严重度 | 位置 | 一句话 |
|---|---|---|---|
| 1 | 致命 | `proposal.md:11-19` | 重复建 `audit_log`，绕过 `0001:411` 与 `0042` 的防篡改实现，重开伪造通道 |
| 2 | 致命 | `proposal.md:23-29` | 裸 `db.prepare().run()` 违反 `tx.ts:7-11`，审计与业务不同事务 |
| 3 | 高 | `proposal.md:12` | 无 `IF NOT EXISTS`、未登记 `REQUIRED_TABLES`，迁移与自检会炸 |
| 4 | 高 | `proposal.md:35` | "保留 6 个月"与 `0042:20` 不可删除触发器直接冲突，清理任务必失败 |
| 5 | 中 | `proposal.md:16` | `ip` 无约束无来源标记，漏配 `DSH_REVIEW_TRUST_PROXY` 时审计失真 |
| 6 | 中 | `proposal.md:21` | "全部写操作记一条"无索引、无容量评估，可拖慢主库 |
| 7 | 中 | `proposal.md:31-32` | 二次确认无失败锁定、失败不审计，可爆破且事后无据 |
| 8 | 低 | `proposal.md` 全文 | 无发布/回滚步骤；加列后不可回滚，需先快照 |

**核心意见**：这个方案看起来是在不了解仓库现状的前提下写的——它要建的东西 4 个迁移编号之前就建好了，还刚做过安全加固（`0042`）。SRE 视角最不能接受的是第 1、2 条：一个**可被删除的审计表**加上**在业务事务之外写的审计行**，会让"等保合规"变成纸面文章，而且上线后无法靠运维手段补救。建议打回，改为"`operation_records` 加一个 `client_ip` 列 + 全量走 `recordOperationInTx`"，方案可从 37 行缩到 10 行，风险和收益都更清楚。