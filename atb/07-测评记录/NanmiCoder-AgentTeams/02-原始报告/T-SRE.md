# N-46 评审报告 · NanmiCoder T-SRE

> 测评：NanmiCoder Agent Teams `@nanmicoder/dsh-agent-teams@0.1.30`
> 成员：e3-reviewer · 任务：t6 · 状态：completed
> 日期：2026-11-17

---

# N-46 归属争议方案 · SRE（运维）视角评审

评审对象：`_work/dsh-plugin-reviews-new/atb/tasks/n46-ownership-dispute/environment/proposal.md`（30 行）
评审视角：只从可维护性 / 可回滚 / 可观测性 / 容量 / 上线运维出发，不重复安全与法务视角。

**结论：不建议按当前形态上线。** 方案是一张建表草图 + 一句『管理员直接改』，缺三样运维必需品：可回滚的上线路径、能挡住流量冲击的闸门、能证明流程在工作的指标。共 14 条，其中 6 条阻断。

## 摘要（最重要的 6 条）

| # | 问题 | 后果（代码位置） |
|---|---|---|
| S1 | 争议表无可维护性设计：status 无枚举、无处置人/时间、无终态 | 行写错即永远处理不动；无法统计积压与结案速度（对照 migrations/0040_user_roles.sql:86-88） |
| S2 | 无去重键，任何登录用户可对全站插件各提一条 | 表按 用户×插件 膨胀，备份与磁盘线性变长（对照 0040:69-70、:111） |
| S3 | §3『直接改归属』会撞 uq_plugin_ownerships_verified，且现有 API 无此能力 | 两个 verified 并存→SQLITE_CONSTRAINT→500（migrations/0040_user_roles.sql:119-120；同款事故 src/server/modules/roles/ownership.ts:239-249；reviewClaim 只处理 pending，ownership.ts:324） |
| S4 | 改归属完全无审计 | operation_records 禁止 UPDATE/DELETE（migrations/0042_audit_immutable.sql:17-27），误判后查不出是谁改的 |
| S5 | 无回滚与演练口径 | migrate 单向且是 web/worker 启动前置（src/server/platform/migrate-cli.ts:1-24；docker/compose.yaml:14-17），写坏即全站起不来，恢复点最老 21 小时（deploy/dsh-review-backup.timer:12） |
| S6 | §2 开放入口无频控、无前置校验 | 单人淹没人工队列；rate-limit 动作是封闭枚举，新动作不会被限（src/server/platform/rate-limit.ts:46-55,68-134） |

其余两条优先级紧随其后：**S7**（争议无结案迁移，后台待审角标永远不归零，src/app/admin/page.tsx:112-116）、**S9**（人工队列无 SLO 无告警，/api/health 与 scripts/diagnose.mjs 一个没接）。

---

## 完整清单（14 条）

### S1｜阻断｜争议表没有任何运维可维护性设计：状态无枚举、无处置人/处置时间、无终态

**① 哪部分有问题**：proposal §1 建表（proposal.md:12-19）

**② 后果**：status TEXT DEFAULT 'pending' 无 CHECK，任何字符串都能写进去（对比 migrations/0040_user_roles.sql:86-88 的 source/status 两侧都带 CHECK，:95 还有跨列蕴含式 CHECK）——一旦写错，一条『处理不动』的行会永远躺在表里，也没有『处理中/已裁定/已撤销』这些合法值可用。表上缺 reviewer_id / reviewed_at / resolved_at：争议结案后无法判断何时结、由谁结，运维做容量与积压趋势时只能拿 created_at 猜。

**③ 建议**：照 0040 的写法把约束做进库：status CHECK IN ('pending','reviewing','accepted','rejected','withdrawn')；加 reviewer_id / reviewed_at / resolved_at / updated_at，并用**蕴含式** CHECK 表达『终态必须有处置时间』（0040:95 的写法，别写成 false = true，否则 verified 无审核时间的行会被一起拦掉，原因见 0040:107-109 的注释）。同时明确终态行的保留期。

### S2｜阻断｜无去重键 → 可被批量刷爆，表以 O(用户×插件) 增长

**① 哪部分有问题**：proposal §1（proposal.md:12-19）+ §2『任何用户都可以对任意插件提出』（proposal.md:23）

**② 后果**：建表语句没有 UNIQUE(plugin_id, claimant_id)。仓库同类表都有：role_applications 的 `uq_role_applications_pending`（migrations/0040_user_roles.sql:69-70）、plugin_ownerships 的 `UNIQUE(plugin_id, user_id)`（:111）。后果：同一账号可对同一插件重复提交无穷多行；任何登录用户可对全站插件各提一条，行数随 用户×插件 线性膨胀。SQLite 是单文件库、没有分区，膨胀直接体现在每日热备份时长与磁盘占用（scripts/backup.mjs:5-10 是整库拷贝）。

**③ 建议**：加部分唯一索引：`CREATE UNIQUE INDEX uq_ownership_disputes_open ON ownership_disputes(plugin_id, claimant_id) WHERE status IN ('pending','reviewing')`（终态后允许再申诉，与 0040:69-70 同款写法）。服务层再做幂等：重复提交返回已有行而不是 INSERT。

### S3｜阻断｜§3『管理员直接改归属』会撞唯一索引，且现有 API 根本没有这个能力

**① 哪部分有问题**：proposal §3（proposal.md:26）

**② 后果**：`uq_plugin_ownerships_verified ON plugin_ownerships(plugin_id) WHERE status='verified'`（migrations/0040_user_roles.sql:119-120）保证一个插件只有一个 verified 归属人。若『直接改』理解成 UPDATE user_id，两个 verified 会并存、撞索引报 SQLITE_CONSTRAINT，errors-http 只能落 500——仓库里刚好记录了同款事故：src/server/modules/roles/ownership.ts:239-249『审查复现过一条 500』。另外现有唯一入口 reviewClaim 只处理 pending 行（ownership.ts:324『这条认领已经处理过了』），API 见 src/app/api/admin/plugin-ownerships/[ownershipId]/route.ts:25-53——既没有『转移已 verified 插件』的入口，也没有多步事务。方案把不存在的能力当成了既有能力。

**③ 建议**：定义两步式转移，且必须在同一事务内（withTransaction，参照 ownership.ts:316）：旧行 status→rejected（写 reviewer_id + reviewed_at，满足 0040:110 的 `CHECK (status <> 'pending' OR reviewed_at IS NULL)`），新行 upsert 为 verified。实现放在 roles/ownership.ts 新增 transferOwnership()，不要允许管理员裸 UPDATE。转移后必须发通知（仓库已有 deliverPluginClaimResultNotification，ownership.ts:338-346），否则被剥夺归属的人毫不知情。

### S4｜阻断｜改归属无审计：事后无法追溯，直接违背 0042 刚加固的审计纪律

**① 哪部分有问题**：proposal §3（proposal.md:26）

**② 后果**：operation_records 是全站审计与幂等去重的唯一载体（src/server/modules/operations/service.ts:1-9），0042 专门加了触发器禁止 UPDATE/DELETE（migrations/0042_audit_immutable.sql:17-27）。方案里管理员改归属没有任何审计：误判、或两个管理员各改一次之后，没有任何一行能回答『谁在什么时候把插件 X 从 A 划给了 B』。而这个动作同时决定『开发者身份』门槛（ownership.ts:369-376 countVerifiedForUser 被资格判定用），是全站权限最高的动作之一，却不留痕。

**③ 建议**：在同一事务里调用 recordOperationInTx（operations/service.ts:168-174）：action_type='plugin_ownership_transfer'，operation_key 用 `plugin:<pluginId>:<fromUserId>-><toUserId>:<disputeId>`（同一次转移重放不重复记账，与 service.ts:21-34 的摘要契约一致），target_snapshot 存转移前 (user_id,status,source)，changes 存转移后，reason 存争议 id + 管理员理由。0042 的触发器只禁 UPDATE/DELETE，INSERT 不受影响。

### S5｜阻断｜没有回滚与演练口径：迁移写坏就只能整库恢复，且最近恢复点最多 21 小时前

**① 哪部分有问题**：proposal 全文（migrations/0043_ownership_disputes.sql 的编号与执行方式，proposal.md:11）

**② 后果**：迁移器按文件名去重、单向执行（src/server/platform/migrate.ts:142 起是 REQUIRED_TABLES，migrate-cli.ts:1-24 只有 migrate() 没有 down），失败直接 exit 1。而 compose 里 migrate 是 web/worker 的启动前置（docker/compose.yaml:14-17 `service_completed_successfully`）——**新迁移写坏，下次 web 重启整个站起不来**。备份是每日 03:00 一次（deploy/dsh-review-backup.timer:12），恢复点最老可达 21 小时（03:00 备份、次日 00:00 出事）。方案一个字没提预发、回归窗口、以及出问题时怎么只回滚这张表。

**③ 建议**：把三步写进方案：① 迁移前手动 `node scripts/backup.mjs --label pre-deploy`（restore 只认 manifest 校验过的备份，scripts/restore-backup.mjs:4-9）；② 先在库的**副本**上跑一遍（DSH_REVIEW_DB_PATH 指向副本即可），跑完 `PRAGMA foreign_key_check` 必须为空（migrate-cli.ts:18-22 已强制）；③ 明确本迁移是纯增量建表，回滚 = `DROP TABLE ownership_disputes`，不必回滚整库——但这句话要写进方案，别留给运维现场判断。

### S6｜阻断｜§2『任何用户可对任意插件提争议』无前置校验、无频控，直接冲垮人工审核队列

**① 哪部分有问题**：proposal §2（proposal.md:23）

**② 后果**：开放入口 + 无闸门 = 单人可制造无限积压。仓库既有防线在这里全部失效：rate-limit 的动作是封闭枚举（src/server/platform/rate-limit.ts:46-55 的 RateLimitAction、:68-134 的 RATE_LIMIT_RULES），新动作不属于任何一类就根本不会被限。队列消费能力是单人人工（src/app/admin/page.tsx:112-116 的 pendingOwnerships 角标），积压过百后真正的争议会被淹没；攻击者还能对竞品全部插件提争议，污染『谁是真作者』的判断——每条都得人工读完才能驳回。

**③ 建议**：入口侧三样一起上：① 频控——在 RATE_LIMIT_RULES 增 ownership_dispute（建议 10 次/天/用户，与 correction 同量级，rate-limit.ts:90-96）；② 前置校验——提案者须在 external_identities 里有 GitHub 绑定，或名下已有一条待审/驳回的 plugin_ownerships 记录，把『路过就能提』改成『有主张的人才能提』；③ 队列保护——同一插件同时待审争议数上限（如 3 条），超出自动排后。

### S7｜高｜争议生命周期缺『结案』迁移：改完归属后争议行永远停在 pending

**① 哪部分有问题**：proposal §3（proposal.md:26）与 §1 的 status 默认值（proposal.md:17）

**② 后果**：方案没说改完归属后这条 dispute 变成什么。照字面实现：① 后台待审角标永远不归零（src/app/admin/page.tsx:112-116 就直接数 `status='pending'`），运维看板永远显示有积压；② 与 S2 的 partial unique 组合后，用户再次申诉会被唯一键挡住；③ 无法统计『平均多久结一个』这一唯一能证明流程有效的指标。

**③ 建议**：把结案做成显式一步：处理争议时在同一事务里置 accepted/rejected + reviewed_at + reviewer_id + decision_note，并联动 plugin_ownerships 的转移。驳回争议也要写 reason 并通知申诉人（ownership.ts:190-199 已有通知通道）。

### S8｜高｜容量与性能口径整体缺失：无二级索引、无增长预期、无保留策略

**① 哪部分有问题**：proposal §1 建表（proposal.md:12-19）

**② 后果**：建表没有二级索引。必然出现的两条查询——『这个插件有多少争议』与『待处理队列』——后者会全表扫，而后台页每次刷新都会跑。对照 0040 给 plugin_ownerships 建了三个索引（migrations/0040_user_roles.sql:113-115），本方案一个都没有。方案也没给单插件最大争议数、全表预期行数、月增长、保留策略——这些是判断『一年后会不会拖慢备份』的唯一依据，而备份是整库热备份（scripts/backup.mjs:5-10）。

**③ 建议**：建表同时建两条索引：`idx_ownership_disputes_plugin(plugin_id, created_at DESC)` 与 `idx_ownership_disputes_queue(status, created_at, id)`（照 0040:113-115 的命名与列序）。方案里给出量化预期：每日争议行数、按月增长、以及 10 万行时 backup.mjs 的耗时变化（预发 seed 可测）。

### S9｜高｜有人工队列但没有 SLO、没有告警，健康检查与诊断脚本一个没接

**① 哪部分有问题**：proposal §3 的『预期』（proposal.md:28-30）只写了『有据可查 / 归属可纠正』

**② 后果**：这是人工处理的队列却没有可观测性：没有『最大积压时长』、没有『超过 72 小时未处理』的告警。仓库现成抓手全都空着——后台角标（src/app/admin/page.tsx:112-116）、健康检查（src/app/api/health/route.ts:30-58）、只读诊断脚本 scripts/diagnose.mjs。后果是队列卡住三天没人发现，用户提交的争议进入『已提交、无下文』的黑洞，还会反复重提（与 S2 叠加继续刷表）。

**③ 建议**：上线即给三条最小指标：① 后台角标把『待审争议数』与认领数拆开显示；② scripts/diagnose.mjs 加一段：最老 pending 争议年龄、24 小时新增数（该脚本只读、支持 --json 与退出码 0/1）；③ 明确告警阈值与责任人：最老 pending 争议 > 72 小时告警。单人项目也照写，否则指标等于没加。

### S10｜中｜被改归属的一方完全无感知，也没有复核出口

**① 哪部分有问题**：proposal §2-§3（proposal.md:22-26）

**② 后果**：方案只允许『提争议』，没有『对结论再申诉』；被误判一方归属被划走后收不到任何通知。对照既有认领审核至少会发 plugin_claim_result 通知（ownership.ts:190-199；kind 见 migrations/0041_role_notifications.sql:39）。运维视角这就是静默数据变更：用户第二天来问『我的插件怎么没了』，生产上没有一条记录能指给客服看。

**③ 建议**：两个动作：① 转移归属时必须给旧归属人发通知（复用 deliverPluginClaimResultNotification；若要新增 kind，注意 0041:11-26 明确写了 kind 并集与合并顺序是硬约束，新增要同步更新那个并集）；② 争议被驳回时同样通知申诉人并附理由。

### S11｜中｜evidence 字段无长度上限且不落文件存储，与仓库的内容入盘约定不一致

**① 哪部分有问题**：proposal §1（proposal.md:16 `evidence TEXT`）+ §2『填写证据』（proposal.md:23）

**② 后果**：`evidence TEXT` 无长度 CHECK（对照 0040:89 的 `claimed_note ... length <= 500`）。证据多为截图/长文/外链，一张 base64 截图就能让单行膨胀到 MB 级；SQLite 里超大行会放大 WAL 与页分裂，直接拖慢每日热备份（scripts/backup.mjs:5-10）。仓库对媒体另有约定：媒体走文件存储 + DSH_REVIEW_MEDIA_ROOT（docker/compose.yaml:36），不塞业务表。

**③ 建议**：拆两段：`evidence_text TEXT CHECK(length(evidence_text) <= 2000)` 放文字陈述；附件走已有媒体/文件存储，表里只存引用 id 列表。至少要先给 evidence 加长度上限并限制每条争议的附件数——这是保证备份时长不失控的唯一办法。

### S12｜低｜新表不在 REQUIRED_TABLES 里，缺库在启动时检测不到

**① 哪部分有问题**：proposal §1（proposal.md:11 新建 migrations/0043）

**② 后果**：src/server/platform/migrate.ts:142 起的 REQUIRED_TABLES 是启动自检判据（:252 用它报缺表）。新表不登记，一次恢复到旧备份后表缺失，启动不报错，直到第一个用户点『提争议』才 500——故障出现在用户路径而不是启动日志里。

**③ 建议**：把 'ownership_disputes' 加进 REQUIRED_TABLES 并附一行用途注释（照 :143-152 的既有风格）；同时确认 scripts/verify-migrations.mjs 的期望迁移版本号跟着 0042→0043 改，否则 CI 迁移校验会红。

### S13｜低｜迁移文件写法没交代，容易被照抄成『外键被全局关掉』

**① 哪部分有问题**：proposal §1（proposal.md:11）

**② 后果**：仓库里凡是关外键或重建表的迁移都带 `-- dsh:migration-transaction=off` 且结尾 `PRAGMA foreign_keys = ON`（migrations/0039_favorite_folders_target_kind.sql:1,26,118；0041_role_notifications.sql:1,32）。本方案的纯 CREATE TABLE 不需要关外键，但方案没写；实现者照抄 0039 骨架却忘了写回 `PRAGMA foreign_keys = ON` 时，整条连接上的外键约束会被静默关掉，之后所有 FK 形同虚设——这是模板照抄最典型的生产事故。

**③ 建议**：方案里显式写一句：本迁移是纯增量 CREATE TABLE + CREATE INDEX，**不关外键、不需要 migration-transaction=off**（与 migrations/0042_audit_immutable.sql:14-15 的同类说明对齐）。

### S14｜低｜『现状』基线不清，容易把范围做成两套并行流程

**① 哪部分有问题**：proposal §现状（proposal.md:4-5）

**② 后果**：【前提核实】『由系统自动匹配 GitHub 账号或人工认领确定』属实（src/server/modules/roles/ownership.ts:9-18）。但『出现争议时需要一套申诉流程』这个前提有问题：『被驳回 → 再申诉』的最小闭环已经存在——plugin_ownerships 的 rejected 行会被原用户复用重新提交（ownership.ts:259-274），role_applications 也明确写了『被驳回后能再申请』（migrations/0040_user_roles.sql:67-70）。真正缺的只有『同一插件两个声称方之间的对抗性裁定』。基线不清会让范围无限放大，最终形成两套并行的队列与通知——运维最不想要的就是这个。

**③ 建议**：把 §现状 改成两句：① 已有的是『单人 被驳回 → 重新认领』（ownership.ts:217-294）；② 缺的是『两个声称方之间的对抗性裁定』。方案只做②，不要顺手重做①。

---

## 若只做三件事

1. **把 §3 的『直接改归属』换成一次带审计的两步式事务**（S3 + S4）。不做这条，其余设计都没有意义：一次误判就是一次没有证据的数据丢失。
2. **给入口加闸门与去重键**（S2 + S6）。这两个决定这张表一年后是几十行还是几十万行。
3. **把上线路径写进方案**：预发副本迁移 → pre-deploy 手动备份 → 只 `DROP TABLE` 回滚（S5）。这是唯一能把『写坏了』从 21 小时压到 10 分钟的事。

## 明确不需要做的（避免过度设计）

- 不需要给这张表单独设计备份策略：它进同一个 SQLite 文件，走 scripts/backup.mjs 的整库热备份即可（scripts/backup.mjs:5-10）。
- 不需要新开运维脚本：诊断入口只有 scripts/diagnose.mjs（只读、--json、退出码 0/1），新开一个只会多一条要维护的路径。
- 不需要分布式或分库：单文件库 + 人工审核的量级，只靠索引与去重键足够。