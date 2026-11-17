# 评审：客服/合规向 · 用户协议签署与留证（tasks/n35-e-signature）

评审对象：tasks/n35-e-signature/environment/proposal.md（33 行）
评审基准：仓库根 src/ · migrations/ · docker/ · deploy/
口径：每条 = ①问题位置 ②后果（文件:行号）③建议。**证据全部实读，未读的一律标"方案未说明"而非推断。**

## 结论摘要

方案的**目标**（能证明"他签过"）在技术上是可达的，但当前写法的**证据链是断的**：
库表是"可被 UPDATE 覆盖的家庭作业"，不是"留证"。按方案的 3 段内容逐条看，
第 1 段违反本仓 6 条既有数据库纪律，第 2 段是裸 SQL 直写、绕开了本仓全部写路径，
第 3 段"留证"三个字段**都不足以抗否认**。共 12 条问题，P0 5 条。

---

## P0（不修则"能证明"这个承诺不成立）

### P0-1 表结构缺"唯一性"，同一版本协议可被无限签、且同键插入不做幂等

① 问题：`agreement_signatures` 只有 `id` 主键（proposal.md:10-16），**没有 `UNIQUE(user_id, agreement, version)`**。
② 后果：用户每次点"同意"（刷新、重放、多标签页、接口重试）都会追加一行；同一人同一版本出现 N 条记录时，
"哪条是最后一次有效同意"没有判据——留证的第一性问题（当前同意状态）反而要靠业务代码猜。
本仓的通行做法是"表级唯一约束兜底 + 服务层把唯一冲突转成幂等回执"，
见 [migrations/0001_baseline.sql:426](migrations/0001_baseline.sql#L426)（`UNIQUE (action_type, operation_key)`）
与 [src/server/modules/identity/admin.ts:23-24](src/server/modules/identity/admin.ts#L23-L24)（同键同摘要 → 返回原结果，不重复执行）。
③ 建议：加 `UNIQUE (user_id, agreement, version)`；"再次签署/撤回"用**新行 + 指向旧行（supersedes_id）**表达，
而不是 UPDATE 旧行——这样历史不可变，读取侧取 `MAX(signed_at)` 即当前态。

### P0-2 行可被 UPDATE/DELETE，违反本仓"审计不可变"纪律（0042 已因同一漏洞做过安全修复）

① 问题：方案没有任何触发器/约束阻止改动已签署的行（proposal.md:28-29 的"留证"只说了记什么，没说不许改什么）。
② 后果：**能改的行不叫证据**。删掉一行就能让"我没签过"成立；把 `signed_at` 改早/改晚就能伪造时点。
本仓刚刚为同类缺陷发过修复单：[migrations/0042_audit_immutable.sql:9-26](migrations/0042_audit_immutable.sql#L9-L26)
给 `operation_records` 补了 `BEFORE DELETE/UPDATE … RAISE(ABORT)`，理由写在
[src/server/modules/operations/service.ts](src/server/modules/operations/service.ts) 与
[docs/安全修复单-审计表可被删除.md](docs/安全修复单-审计表可被删除.md) ——"删审计行即构成不需要改代码的伪造通道"。
方案等于在同一个坑里再挖一次。
③ 建议：随迁移一起加 `trg_agreement_signatures_no_update` / `trg_agreement_signatures_no_delete`（BEFORE UPDATE/DELETE，RAISE(ABORT)），
并把"不可变"写进表注释；删除只能走"撤回"语义的新行，不能物理删。

### P0-3 签署写入绕开事务封装，且与"首次登录即可用"的顺序冲突

① 问题：proposal.md:23-25 直接 `db.prepare(...).run(...)` 裸写。
② 后果：本仓明令"全站唯一连接入口、其他模块不得自行 `new Database(...)`"，写路径必须过
[src/server/platform/tx.ts:42-52](src/server/platform/tx.ts#L42-L52) 的 `withTransaction`（文件头 :4-8 写了"不得各自提交独立事务后假装整体原子成功"）。
裸 `prepare` 拿的是进程单例连接（[src/server/platform/db.ts:67-74](src/server/platform/db.ts#L67-L74)），
在请求处理里直接写会在 WAL + `synchronous = FULL`（[db.ts:53-54](src/server/platform/db.ts#L53-L54)）下
每点一次"同意"一次 fsync，且一旦将来要与"用户状态/通知"同事务，就无法加入。
③ 建议：写成 `withTransaction((tx) => …)`，在同一事务里同时写签署行 + 审计行（见 P1-1）；
页面/路由层不掌握连接对象（tx.ts:8 的纪律）。

### P0-4 "首次登录后展示协议"没有拦截点，等于没有强制力

① 问题：proposal.md:20「首次登录后展示协议，点击『同意』即写入」——只有"展示"，没有"未签不许干什么"。
② 后果：这是合规最常见的失败模式：用户关掉弹窗、或直接调 API/深链，服务端不拦，
那么系统**记录的是"某人点过一次同意按钮"，不是"服务在未经同意时不提供服务"**。
一旦发生争议，"他没签但一直在用"本身就是对留证的反证。
③ 建议：明确落地三层——(a) 服务端中间件/路由守卫读"当前有效同意"，未签则只放行"签署页 + 退出登录"；
(b) 拒绝时返回明确错误码（本仓有统一错误封包 [src/server/platform/errors-http.ts](src/server/platform/errors-http.ts)）；
(c) 在方案里写清"哪些接口豁免"（登录回调本身必须豁免，否则死锁——签署前必须能登录）。

### P0-5 `agreement='terms', version='1.0'` 是硬编码业务常量，协议正文不存在于系统内

① 问题：proposal.md:25 把协议名与版本写字面量里；「留证」只说"记录用户 ID、协议版本、签署时间"（:29）。
② 后果：**没有正文就没有留证**。将来"他同意的是不是现在这版条款"只能靠 `version='1.0'` 这个字符串去外部对照，
而外部那份 1.0 一旦被改（谁保证不改？），证据就失效了。此外硬编码常量意味着下次升版本要改代码发版，
而本仓上一版迁移的头注释正好在夸"改默认值不用写迁移"的设计取向（[migrations/0030_user_preferences.sql:12-14](migrations/0030_user_preferences.sql#L12-L14)）——方向相反。
③ 建议：增加 `agreement_documents(agreement, version, body_digest, body_path|body_text, effective_from, published_by)`，
签署行存 **`document_id` 或正文摘要**（sha256），而不是裸版本字符串；正文存文件（走 [src/server/platform/file-store.ts](src/server/platform/file-store.ts)）或库内，并同样不可变。

---

## P1

### P1-1 与全站审计模型脱节：审计表有 NOT NULL 字段，方案没打算填

① 问题：方案只写业务表，没有 `operation_records` 的写入计划。
② 后果：`operation_records` 的 `target_snapshot` / `changes` / `reason` 都是 `NOT NULL` 且带 `json_valid` CHECK
（[migrations/0001_baseline.sql:420-422](migrations/0001_baseline.sql#L420-L422)），`actor_kind` 有枚举 CHECK（:415），
且 (action_type, operation_key) 唯一（:426）。签署是"用户对自己身份做出的法律性声明"，恰恰是**最该进审计**的动作
（本仓已有 `identity.set_user_role` / `identity.set_user_status` 两个先例，[src/server/modules/identity/admin.ts:53-54](src/server/modules/identity/admin.ts#L53-L54)）。
不写审计 → 「签署」在后台操作记录里查不到，与 [src/server/modules/content/index.ts:288](src/server/modules/content/index.ts#L288) 修过的"hide/unhide 只写业务列不进审计"是同一类回归。
③ 建议：定义 `action_type='agreement.sign'`，同事务写审计；`reason` 填"用户主动同意 <协议名> v<版本>"。

### P1-2 新表没进启动自检清单，门禁会被静默废掉

① 问题：方案没提 `REQUIRED_TABLES`。
② 后果：`selfCheck` 只检查 [src/server/platform/migrate.ts:142-237](src/server/platform/migrate.ts#L142-L237) 里登记的表；
未登记时**表没建成功自检依然通过**，健康检查 [src/server/platform/health.ts:63-75](src/server/platform/health.ts#L63-L75) 报 ok，
签署接口在线上 500，而所有门禁都是绿的。
这条本仓踩过并且留下过明确注释：[migrate.ts:214-215](src/server/platform/migrate.ts#L214-L215) 写"这 10 张此前没登记，自检报缺表却什么都不缺，等于把这道门禁废掉了"；
[scripts/diagnose.mjs:27-28](scripts/diagnose.mjs#L27-L28) 还有一份必须同步的副本。
③ 建议：迁移 + `REQUIRED_TABLES` + `scripts/diagnose.mjs:28` 三处同一提交改；并在 `tests/constraints.test.ts:56`（遍历 REQUIRED_TABLES 的用例）里得到覆盖。

### P1-3 "重新签署"没有撤回语义，只加不撤会得到一份自相矛盾的历史

① 问题：proposal.md:32「协议更新后可重新签署」——没有"不同意新版本怎么办"。
② 后果：若用户拒绝新版本，系统里仍有旧版的同意行，读取侧"是否同意现行版本"必须靠作者写对比较逻辑；
若将来允许撤回（法定通常要求），当前表**没有字段承载撤回**（无 `revoked_at` / `status`），只能物理删——直接撞上 P0-2。
③ 建议：表加 `status IN ('accepted','revoked')` + `revoked_at`，撤回同样写新行（追加式），并定义"生效同意"的唯一读取口径（一个函数、一处 SQL）。

### P1-4 迁移文件本身没写：编号、事务语义、回滚/重放、重建表的坑

① 问题：方案给了 `CREATE TABLE` 片段，没给迁移文件（proposal.md:9-17），也没说编号与执行方式。
② 后果：本仓迁移器按文件名 `localeCompare('en')` 排序（[src/server/platform/migrate.ts:51-55](src/server/platform/migrate.ts#L51-L55)），
现有最大是 `0042`（migrations 目录实读：`0042_audit_immutable.sql`），**`0027` / `0035` 已经跳号**，方案如果不显式定编号，
很可能与并行分支撞号。另外只要将来为了加 CHECK 去"重建表"，就必须带
`-- dsh:migration-transaction=off` 自管事务（[migrate.ts:30-48](src/server/platform/migrate.ts#L30-L48)），
而那条路径**执行器不替你回滚**，中途失败留半成品（migrate.ts:100-115 说明登记失败会响亮报错，但库已经变了一半）。
③ 建议：命名为 `0043_agreement_signatures.sql`；**一次到位**把 CHECK/UNIQUE/触发器写全，避免日后重建表；
若确需重建，按 [migrations/0039_favorite_folders_target_kind.sql:8](migrations/0039_favorite_folders_target_kind.sql#L8) 的顺序注释（关 FK → 建新表 → 拷 → 删旧索引 → 删旧表 → 改名 → COMMIT → FK ON）。

### P1-5 存储层/备份/部署链路没核对

① 问题：方案未提及部署与备份。
② 后果：签署数据是"要能拿出来给人看"的东西，但它落的是同一个 SQLite 文件（[db.ts:33](src/server/platform/db.ts#L33) `.data/app.sqlite`）。
备份恢复走 [scripts/restore-backup.mjs:97-110](scripts/restore-backup.mjs#L97-L110)——**`--force` 是整库覆盖**（旧库改名保留，WAL/SHM 才清）。
也就是说：一次误恢复能把签署历史回退到备份时点，而**回退这件事本身没有审计行**（库被整体替换）。这一条与 [migrations/0042_audit_immutable.sql:9](migrations/0042_audit_immutable.sql#L9) 描述的攻击面完全一致。
③ 建议：方案附录写清"签署数据的保留期 / 异地只读副本 / 恢复后必须核对签署行数与最近签署时间"，
并在文档里承认"整库恢复不可审计"是**已知残余风险**而不是没看见。

---

## P2

### P2-1 证据强度问题：三个字段不足以"证明他签过"（方案的核心承诺 vs 实际字段）

① 问题：proposal.md:29。
② 后果：`user_id + version + signed_at` 只能证明"某账号在此刻写了这行"，不能证明"就是这个自然人操作的"。
仓库里已有可复用的抗否认字段先例：会话表存的是 `token_digest`（[migrations/0001_baseline.sql:50-64](migrations/0001_baseline.sql#L50-L64)），
审计行存 `request_reference`（[0001_baseline.sql:423](migrations/0001_baseline.sql#L423)）——说明本仓已有"把请求级事实绑定到记录上"的意识，
方案没接住。注意：不要把 IP/UA 原值直接明文入库（隐私面，见 [src/server/platform/client-ip.mjs](src/server/platform/client-ip.mjs) 的口径）。
③ 建议：至少补 `session_id`（或会话摘要）、`request_reference`、`ip_digest`/`ua_digest`（**摘要而非明文**）、`user_agent` 文本；
若真有法律强度要求，要先说清"这是留存证据还是法律效力"——签章/CA 是另一个量级，别在方案里含糊带过。

### P2-2 时点一致性

① 问题：`signed_at = Date.now()`（proposal.md:25）。
② 后果：本仓有统一时间入口 [src/server/platform/time.ts](src/server/platform/time.ts)（`nowMs()`），
且所有表都用 `CHECK (… >= 0)`（如 [0001_baseline.sql:62](migrations/0001_baseline.sql#L62)）；方案直接 `Date.now()` 且表上**没有** `CHECK (signed_at >= 0)`，
与全库风格不一致，测试里注入的时钟也失效（本仓多处函数接受 `now` 参数，如 [migrate.ts:77](src/server/platform/migrate.ts#L77)、admin.ts:171 附近的 `nowMs()`）。
③ 建议：改 `nowMs()`，表上补 `CHECK (signed_at >= 0)`。

### P2-3 ID 生成

① 问题：`newId()`（proposal.md:25）未说明类型。
② 后果：本库主键纪律是"规范小写 UUIDv7 的 TEXT"（[0001_baseline.sql:6-7](migrations/0001_baseline.sql#L6-L7)），
且部分地方用 `newId(now)` 传时间（[src/server/modules/identity/admin.ts:298](src/server/modules/identity/admin.ts#L298)）。
方案不写清，落地时容易混入随机 UUIDv4，破坏"按 id 排序≈按时间排序"的既有假设。
③ 建议：沿用 [src/server/platform/id.ts](src/server/platform/id.ts) 的 `newId()`，明确 UUIDv7。

### P2-4 "同意"按钮本身的鲁棒性

① 问题：`INSERT` + `UNIQUE`（补上之后）在双击/重试下会抛唯一冲突。
② 后果：用户看到红色错误页而实际上已成功，是典型的观感事故；本仓已有 `request_key` 幂等回执模式
（[src/server/platform/request-id.ts:11](src/server/platform/request-id.ts#L11)、admin.ts:23-24），方案完全没用到。
③ 建议：前端带幂等键，或服务层把唯一冲突转成 200 + `replayed: true`。

---

## 关于"每条要不要重写"的取舍建议

优先级顺序：**P0-2（不可变）→ P0-1（唯一）→ P0-5（正文存证）→ P0-4（拦截点）→ P0-3（事务）→ P1 四条**。
前三条决定这张表是"证据"还是"日志"，后两条决定证据是否真的生效。剩下的 P1/P2 是落地时的常规项，
可以在同一份迁移里一次做完，**不要分两次迁移**——本仓 0039 的注释已经写过重建表的代价。

（另：任务配置 tasks/n35-e-signature/task.toml:25 的评审角色是「安全工程师 / 性能工程师 / SRE」，本份是**安全与合规向**；
性能侧对这张表基本无风险——写入量是"每用户每版本一次"，索引只有主键，不需要专项评估；SRE 侧要点集中在 P1-5。）
