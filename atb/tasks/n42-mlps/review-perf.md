# 评审：等保二级合规改造 · 性能工程师视角

> 评审对象：`atb/tasks/n42-mlps/environment/proposal.md`（37 行）
> 评审人身份：**性能工程师**（只看写放大、fsync 成本、表增长曲线、查询路径与索引、保留期带来的备份成本；不评合规条款措辞）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**§2「所有写操作都记一条」会把全站写请求再加一次 fsync，而这条日志没有任何读路径、没有索引、没有保留期的执行者**——
> 等于用「每个写请求多一次同步提交」换一张只会长大的表。而且仓库**已经有一张不可变审计表**（`operation_records`），方案是第二套。

---

## 摘要（前 5 条最重要）

1. 【严重】**§2 的写法把「审计」从「已有 28 处调用点、与业务同事务」变成「每个写都多一次独立 INSERT」。**
   依据：`proposal.md:21-27`（`db.prepare(...).run(...)`，**没有事务句柄参数**、没有复用 `recordOperationInTx`）；
   仓库现状：`src/server/modules/operations/service.ts:161-174` 提供同步的事务内入口，全仓 **28 处** `recordOperationInTx(...)` 调用点 + 3 处内联 INSERT（`service.ts:139`、`identity/admin.ts:292`、`catalog/service.ts:545`）。
   后果：方案里的 INSERT 若在业务事务**之外**执行，就是「业务提交了但审计没写进去」或反之——而这恰恰是 0042 立规要堵的东西；
   若在事务内执行，则**与现有 `operation_records` 双写**：每次写操作两次审计 INSERT、两次索引维护。
   建议：不建新表（见摘要 2），把等保要的字段补进 `operation_records` 的既有列（`target_snapshot / changes` 是 JSON，能装 IP 与客户端信息）。

2. 【严重】§1 新建 `audit_log` = 第二套审计，且**没有 0042 那样的不可变守卫**。
   依据：`proposal.md:9-18`（只有 PK，无触发器、无索引）；`migrations/0001_baseline.sql:411-431`（`operation_records` 已有 15 列含 `request_reference`、`target_snapshot`、`changes`、`task_id`）；
   `migrations/0042_audit_immutable.sql:17-27`（`trg_operation_records_no_delete/no_update`，`RAISE(ABORT)`）。
   性能后果：两套表意味着**两套增长曲线 + 两套索引**（`operation_records` 已有 `idx_operation_target`、`idx_operation_actor`，`0001:432-433`），
   而 `audit_log` 一条索引都没有 → 将来管理员要查「某用户今天做了什么」就是全表扫；
   `sqlite` 单文件、`synchronous = FULL`（`src/server/platform/db.ts:54`）下，多出来的这部分写入是纯增量成本。
   建议：删掉 `audit_log`，把等保要求的「IP」作为一列加到 `operation_records`（或塞进既有 `request_reference`）；
   若坚持独立表，必须同步加不可变触发器与至少一条 `(created_at)` 索引。

3. 【严重】§4「保留 6 个月」在 `synchronous = FULL` + 单文件 SQLite 下是**可量化的磁盘与备份成本，方案没有算**。
   依据：`src/server/platform/db.ts:52-56`（WAL + `synchronous = FULL`，每次提交 fsync）；
   `scripts/backup.mjs:78-94`（备份会对 `operation_records` 等关键表做行数对账）、`:148-157`（`--keep 14` 按目录名保留 14 份）；
   `deploy/dsh-review-backup.timer:12`（每天 03:00 一次全量备份）。
   后果：审计表是全库增长最快的表（每个写操作一行）。6 个月后它的体积会主导整个 `app.sqlite`，
   于是**每天一次的全量备份、每份 14 天保留 = 14 × 一个被审计表撑大的库**；`backup.mjs` 里对它的 `count(*)` 也会变慢。
   而 `operation_records` 是**不可删除**的（0042 触发器），意味着「保留 6 个月」在现有审计表上根本无法执行——
   方案要么接受永久保留，要么必须**先设计归档导出**（把老行导出去再删），而 0042 明确禁止 DELETE。
   建议：在方案里给出量化口径：日均写操作数 × 6 个月 ≈ 多少行 / 多少 MB；并明确「审计归档」是独立流程（导出到文件后由运营侧留存），
   不能指望库内 DELETE。

4. 【高】没有读路径设计：审计表存在的意义是「被查」，而方案没给任何查询场景、索引或访问接口。
   依据：`proposal.md` 全文只有 INSERT；现有审计的读路径是 `src/app/admin/operations/page.tsx:118-166`：
   `:118-126` **无 WHERE 的全表 GROUP BY**（操作人下拉）、`:128-132` 又一条**全表 GROUP BY action_type**、`:150-152` 的 `COUNT(*)` 同样全表。
   后果：如果新表也走「后台页全表聚合」，那么表越大后台越慢（且这三条查询在 `better-sqlite3` 同步驱动上直接阻塞事件循环，`db.ts:6-8`）；
   如果没有读路径，这张表就只是磁盘负担。
   建议：把等保要的「审计查询」直接落在现有表格上：给 `operation_records(created_at)` 或 `(action_type, created_at)` 补一条索引，
   并把后台那两条 GROUP BY 改成 `LIMIT 50` + 短 TTL 缓存（这正是同仓 m 系列题目里已经点出的同一处慢点）。

5. 【高】§2 的 `Date.now()` 与 `newId()` 直调，和仓库的「事务内同步、时钟可注入」纪律冲突，且会让批量写入放大成 N 次 `prepare`。
   依据：`proposal.md:24-27`（每次调用都 `db.prepare(...)`，**语句没有预编译复用**）；
   `src/server/platform/time.ts:1-13`（「业务代码不得散落 `Date.now()`」，且允许注入 `now` 以便测试）；
   `src/server/modules/operations/service.ts:71-95`（服务层已封装参数校验与 `now` 注入）。
   后果：批量导入/发现这类一次写几十上百条的场景，会变成 N 次 `prepare + run`，每次一次 fsync；
   在 `synchronous = FULL` 下，10 条审计 ≈ 10 次 fsync，单条可达毫秒级。
   建议：若必须落表，用**一个事务 + 预编译一次语句**（`const stmt = db.prepare(...)` 提到循环外），并把事务边界与业务写对齐。

---

## 完整问题清单（按严重度）

### P-01 【严重】每个写操作多一次独立 INSERT（无事务绑定）
- 现场：`proposal.md:21-27`。
- 依据：`src/server/modules/operations/service.ts:161-174`（事务内同步入口）、`:71-95`（参数校验 + `now` 注入）、`:97-134`（同键幂等）；全仓 28 处调用点。
- 后果：见摘要 1。补充一个具体数字：`synchronous = FULL`（`db.ts:54`）意味着**每次事务提交一次 fsync**；
  独立 INSERT 自成事务 ⇒ 每个写请求多一次 fsync，NVMe 上约 0.1-1ms、机械盘/云盘上 5-10ms。
- 建议：不建新表；若保留，必须与业务同事务（否则审计可能与业务不一致，且多一次提交）。

### P-02 【严重】第二套审计表，无不可变守卫、无索引
- 现场：`proposal.md:9-18`。
- 依据：`migrations/0001_baseline.sql:411-431,432-433`；`migrations/0042_audit_immutable.sql:17-27`。
- 后果：见摘要 2。
- 建议：复用 `operation_records`；等保需要的 IP/客户端信息进 `request_reference` 或新增一列。

### P-03 【严重】6 个月保留期与「全量备份 × 14 份」叠加，磁盘成本未评估
- 现场：`proposal.md:32-33`。
- 依据：`src/server/platform/db.ts:52-56`；`scripts/backup.mjs:78-94,148-157`；`deploy/dsh-review-backup.timer:12`；`0042:17-27`（审计不可删）。
- 后果：见摘要 3。
- 建议：给出「行数 × 字节」的容量推演；归档走导出而非 DELETE；明确备份保留期与审计保留期的关系。

### P-04 【高】审计读路径缺失；现有后台查询本身就是全表聚合
- 现场：`proposal.md` 无读路径。
- 依据：`src/app/admin/operations/page.tsx:118-132,150-166`；`db.ts:6-8`（同步阻塞）。
- 后果：新表要么永远没人查（纯成本），要么被套进同样低效的全表聚合。
- 建议：审计查询走「带索引 + LIMIT + 缓存」；补 `operation_records(created_at)` 类索引。

### P-05 【高】无预编译语句、无批量写入，批量场景下 fsync 次数 = 行数
- 现场：`proposal.md:24-27`。
- 依据：`src/server/platform/db.ts:52-56`；`src/server/platform/time.ts:1-13`。
- 后果：见摘要 5。
- 建议：语句预编译提到循环外；批量写合并成一个事务；时间戳统一从 `nowMs()` 取并注入。

### P-06 【中】`ip` 列把可变长度字符串塞进最热的写路径
- 现场：`proposal.md:15`（`ip TEXT`）。
- 依据：仓库对「用户可写文本」一贯加长度 CHECK（`migrations/0040_user_roles.sql:55,57`）；`src/server/platform/client-ip.mjs` 显式受 `DSH_REVIEW_TRUST_PROXY` 影响（`docker/compose.yaml:59`）。
- 后果：无长度上限意味着任意长度的头值都能进表（可被用来放大行体积）；而在 SQLite 里行越大，B-tree 页分裂与备份体积都跟着涨。
- 建议：`ip TEXT CHECK (ip IS NULL OR length(ip) <= 45)`（IPv6 最长 45 字符）；并在写入前归一化（去掉端口、去括号）。

### P-07 【低】没有「审计写入耗时」的可观测
- 现场：`proposal.md:35-37`。
- 依据：`server.mjs:62-82`（每请求一行含 `duration_ms`，超过 `DSH_REVIEW_SLOW_REQUEST_MS` 默认 1000ms 会打 warn）；`docker/compose.yaml:57-59`。
- 后果：审计带来的延迟增量只会体现为「整体变慢」，无法归因。
- 建议：上线前后对比 P50/P95 的 `duration_ms`（已有日志足够），必要时把「审计 INSERT 耗时」单独计时打点。

---

## 结论与建议

**方案在性能上不可按原样落地**：它把「已有 28 处事务内审计」变成「每个写都多一次独立 INSERT」，
而新增的表没有索引、没有查询场景、没有保留期的执行者，却要把 6 个月的数据与 14 份全量备份叠在同一个 SQLite 文件上。

**建议的三步改造**：
1. **不建新表**：等保要求的 IP/客户端信息加进 `operation_records`（优先放 `request_reference`，不够再加列），
   写入统一走 `recordOperationInTx`（`src/server/modules/operations/service.ts:161-174`），与业务同事务。
2. **补索引与读路径**：`operation_records(created_at)`（或 `(action_type, created_at)`）；后台聚合加 `LIMIT` + 缓存。
3. **把保留期写成可执行的运维流程**：审计不可删（0042）⇒ 归档 = 定期导出到文件 + 运营侧留存，而不是 DELETE；
   并在方案里给出容量推演（日均写操作数 × 6 个月 → 行数/MB）与备份保留期（`backup.mjs --keep 14`）的关系。

> 评审人声明：本报告只覆盖性能/容量/IO 视角。等保 2.0「审计记录应受保护」「二次密码确认」等条款的合规判定、
> 以及本项目全站 GitHub OAuth（无密码）与「二次密码」的冲突，属于合规/安全视角，本文不代替它们的结论。

> 前提核实：方案 §1 的 `audit_log` 是**声明新建**的表（合理），但其「重复建设」的对象 `operation_records` 真实存在
>（`migrations/0001_baseline.sql:411-431`）且已加固（`0042`）；§2 引用的 `clientIp` 在仓库中确有实现
>（`src/server/platform/client-ip.mjs`，受 `DSH_REVIEW_TRUST_PROXY` 控制）；§2 的 `newId()`/`Date.now()` 都存在，但与 `time.ts:1-13` 的纪律不符。
