# N42 等保二级合规改造 · 安全工程师评审

**评审对象**：`atb/tasks/n42-mlps/environment/proposal.md`（全文 37 行）
**仓库事实核对**：`migrations/`、`src/`、`scripts/`、`docker/` 已逐条落到行号。
**结论**：**不通过**。方案的安全主线建立在三个不存在的假设上（没有密码、没有第二套审计的必要、IP 不可信），且**新建的 audit_log 比现有审计链更不合规**。

---

## 一、【P0·阻塞】§1 新建 `audit_log` 是"安全退坡"：它把已经建好的防篡改审计链绕开

**① 哪部分**：proposal.md:9-18（`CREATE TABLE audit_log`）

**② 后果（文件:行号）**：
- 本仓**已有**全站唯一审计载体 `operation_records`：`migrations/0001_baseline.sql:411-433`（含 `actor_kind IN ('user','admin','system')` `:415`、`actor_user_id REFERENCES users(id)` `:416`、`UNIQUE(action_type, operation_key)` `:426`、两个索引 `:432-433`、json_valid 约束 `:420-421`）。
- 且**已经加了不可变守卫**：`migrations/0042_audit_immutable.sql:17-21`（`trg_operation_records_no_delete`）、`:23-27`（`trg_operation_records_no_update`）。该迁移注释自述其存在理由就是"删审计行 + restore --force 构成不需要改代码的伪造通道"（`:11-12`）。
- 方案的新表**没有任何触发器**。任何能写库的人可 `DELETE FROM audit_log` 或 `UPDATE ... SET user_id=NULL`。
- 这直接违反等保 2.0「安全审计」的**审计记录应受保护，避免未预期的删除、修改或覆盖**——用新表替换，等于**在合规检查上主动降级**。方案 §1 甚至没有意识到 0042 的存在。

**③ 建议**：删除 §1 与新表。改为对 `operation_records` 做**增量补齐**（IP 放进 `request_reference` 或新增列），并写一份"应审计但尚未审计的写路径清单"逐条落账（现有调用点 27 处 / 11 个文件，如 `src/server/modules/identity/admin.ts:256,292`、`src/server/modules/catalog/service.ts:333`）。

---

## 二、【P0·阻塞】§2「所有写操作都记一条」只覆盖模型层，**会丢审计并引入伪造通道**

**① 哪部分**：proposal.md:21-27（裸 `INSERT INTO audit_log`）

**② 后果（文件:行号）**：
- **审计与业务不同事务**：仓库既定纪律是"业务变更与审计在同一事务内写一条"（`src/server/modules/identity/admin.ts:23-27` 明文说明，并解释为何必须内联 INSERT 而不能走 async 的 `record()`）。方案的裸 INSERT 没有说明事务边界——一旦业务回滚而审计留下，审计将记录**未发生的事**；反之业务提交而审计丢失，则无据可查。
- **无幂等键 → 无防伪**：现有审计依赖 `UNIQUE(action_type, operation_key)`（`0001:426`）把"重放"与"重做"区分开。`audit_log` 只有 `id TEXT PRIMARY KEY`（proposal.md:12），同一次操作被重试会写成两条，攻击者亦可批量灌垃圾行稀释真实记录。
- **无法表达系统动作**：`user_id TEXT` 可为 NULL（proposal.md:13），而现有表用 `actor_kind` 三态 + CHECK 强制约束（`0001:415,427-430`）。本仓大量关键动作由系统发起（`src/worker/scheduled.ts:106`、`src/server/modules/catalog/discovery.ts:86`），在新表里全部塌成一个 `NULL`，"系统干的"与"来源不明"无法区分——**审计失去归因能力**。
- **无签名/无哈希链**：两条审计之间没有链接关系，被删一行不留痕。0042 的触发器至少让"删除"会抛错；新表连这个都没有。

**③ 建议**：审计写入统一走 `operation_records`（沿用既有 `operations.service` 与内联两种写法），并为每条记录补 `request_reference`（含 request_id + 客户端 IP）。若确需"对外可查审计"，做**只读视图 + 管理员查询接口**，不要再建表。

---

## 三、【P0·阻塞】§3「管理员操作需要二次密码确认」——**本仓库没有密码**

**① 哪部分**：proposal.md:30

**② 后果（文件:行号）**：
- 全站登录只有 GitHub OAuth：`src/server/modules/identity/service.ts:1-16`（会话令牌，明文凭证不入库，库里只存 `digestText`），身份绑定在 `external_identities`（`service.ts:53-56`）。
- 全仓 `src/` 内 `password` 只出现在两处：日志脱敏正则 `src/server/platform/log.ts:24`、以及检测配置模板里硬编码密码的规则 `src/server/modules/packs/index.ts:415`。**没有任何密码哈希实现**（无 scrypt/argon2/bcrypt 调用）。`src/server/modules/identity/` 下四个文件均无 password 相关代码。
- 因此 §3 是**不可实现的**。若强行"引入密码"，则是给"只有 OAuth、无密码泄露面"的系统**新增一个凭据面**：需要哈希参数、重置流程、锁定策略、泄露检测——这是本方案完全没写、却会把整个攻击面放大的改动。

**③ 建议**：改为与现有架构自洽的**再认证（step-up）**：
- 推荐：**重走一次 OAuth**（把现有 GitHub 回环复用为 `prompt=login` 的再认证，绑定到当前 session_id + 有效期 ≤ 15 分钟），效果等价于"二次确认"且不新增凭据。
- 备选：**双人复核（four-eyes）**——对高风险操作（改角色、禁用账户、恢复备份）要求第二名在任管理员在 30 分钟内确认；这与既有的"至少保留一名可用管理员"守卫（`src/server/modules/identity/admin.ts:13-14,186-190`）同一思路，成本最低。
- **若仍要密码**，必须在方案里给出：scrypt 参数、失败锁定（复用 `src/server/platform/rate-limit.ts`）、重置流程、以及"密码泄露后如何吊销全部会话"。缺一不可。

---

## 四、【严重】§2 的 `clientIp` 是从请求头取的明文 IP——可伪造、且违反最小化

**① 哪部分**：proposal.md:26（`clientIp` 变量在方案中**来源未定义**）

**② 后果（文件:行号）**：
- 仓库对此有专门守卫模块：`src/server/platform/client-ip.mjs:54-71`——**只有 `trustProxy` 为真时才读 `X-Real-IP`/`X-Forwarded-For`**，注释 `:7-11` 明写"直接取 x-forwarded-for 等于让客户端自己决定日志里记的是谁"。方案对此毫不知情；若实现者顺手 `req.headers['x-forwarded-for']`，审计里的 IP 就**可被攻击者任意指定**，等于把溯源证据交给被审计对象。
- 部署侧：`docker/compose.yaml:57-59` 已把 `DSH_REVIEW_TRUST_PROXY=1` 写死，且 `:63` 注明"刻意不发布端口，对外只有 nginx"。**任何绕过 nginx 直连 3000 的路径都会让该值瞬间变成可伪造**。方案没有把这个前提写成约束。
- worker 进程不经 HTTP（`compose.yaml:78+` 独立服务），系统动作的审计行**根本拿不到 IP**，该列会出现"访客 IP / 反代 IP / NULL"三种语义混杂。
- IP 属个人可识别信息：明文长期落库（方案 §4 保留 6 个月）且无粒度收敛，超出审计所需的最小必要。

**③ 建议**：① 必须显式调用 `resolveClientIp({headers, socketAddress, trustProxy})`，**禁止自取请求头**；② `trustProxy` 取值必须来自环境且上线前核对 nginx 覆盖式写入（方案要写进部署清单）；③ 落库按下文 §五 做 `/24`（v4）/`/48`（v6）收敛，或直接存加盐哈希；④ 系统动作显式标注 `actor_kind='system'` 并允许 IP 为空，不要用 NULL 冒充。

---

## 五、【严重】保留 6 个月 ≠ 可恢复：审计与备份轮转存在**证据灭失窗口**

**① 哪部分**：proposal.md:32-33（"保留 6 个月"）

**② 后果（文件:行号）**：
- 备份只留 14 份：`scripts/backup.mjs:13,146,176`、`docker/compose.yaml:62`（`BACKUP_KEEP: '14'`）。日志若只在库里，一旦库损坏，**恢复后最老只剩约 14 天**——那 6 个月里剩下的 5 个多月证据**不可恢复**。
- 更关键：`scripts/restore-backup.mjs:97-110` 的 `--force` 是**整库覆盖**（旧库 `renameSync` 改名保留后 `copyFileSync` 覆盖，并清 `-wal/-shm`）。恢复动作本身会**整段抹掉当前库里的审计**，而"谁执行了恢复、为什么恢复"恰恰是最该有记录的事件——现有 `src/app/api/admin/backups/route.ts` 也仅提供备份，**没有审计恢复操作**。
- 方案没有定义：6 个月后**如何删除**（只在库里删会与 0042 的 `BEFORE DELETE` 触发器冲突：`0042:17-21` 会直接 `RAISE(ABORT)`，**保留期清理与不可变约束正面打架**）、是否归档外置、以及归档的完整性校验。

**③ 建议**：① 审计必须**独立于业务库的备份轮转**做归档（导出为带 sha256 的只追加文件，例如每日导出到 `/app/backups/audit-archive/`，独立于 14 份轮转）；② 明确"保留期到期删除"的实现路径——**fork/导出式删除**（建新表→导 6 个月内的→替换），而不是对 0042 保护的表直接 DELETE；③ 把 `restore-backup --force` 本身列为**必须审计并双人复核**的高危操作（见 §三）。

---

## 六、【严重】§3 未覆盖"审计记录的访问控制"，而这是等保二级的硬条款

**① 哪部分**：proposal.md:3-4、:36（只声明"满足等保二级审计要求"）

**② 后果（文件:行号）**：
- 等保要求「审计记录只能被授权用户访问」。方案**没有任何读取面设计**：新表建好后，谁能查、通过什么接口查、查询行为本身是否被审计——全部空白。
- 现状对照：后台已有审计只读页 `src/app/admin/operations/page.tsx:5,121,130,151,159`，但它读的是 `operation_records`；新表若建立，**这个页面看不到新表的内容**，形成"两套账、两个查询面"。而诊断脚本 `scripts/diagnose.mjs` 同样不涉及审计访问。
- "谁看了审计"如果不留痕，管理员本身的窥探行为就是不可审计的——这本身也是等保要防的。

**③ 建议**：在方案里补齐 (a) 只读查询面（管理员 only，`assertCanAdmin`，分页 + 时间窗强制）；(b) **审计的审计**：任何审计读取也写一条 `operation_records`；(c) 导出/打印权限单列并双人复核。

---

## 七、【中】"二次确认"缺少抗暴力破解设计（若按密码方案实现）

**① 哪部分**：proposal.md:30

**② 后果（文件:行号）**：
- 现有频控体系**完全未覆盖登录/鉴权类动作**：`src/server/platform/rate-limit.ts:46-55` 的 `RateLimitAction` 九个动作全是业务写（评论/提交/举报/纠错/反馈/论坛/申请），**没有 `admin_reauth` 之类**。且 `enforceRateLimit` 的键是 `<动作>:<userId>`（`:226`），按**用户**计数——对"被劫持管理员会话"的暴力猜测**天然无效**（攻击者就是那个用户）。
- 因此若照方案实现"二次密码"，就是一个**无频率限制的在线口令校验点**。

**③ 建议**：新增 `admin_reauth` 动作，但键必须是 `userId + clientIp` 复合（不是纯 userId），并叠加**指数退避 + 连续失败 N 次即全局吊销该用户所有会话**（`identity/service.ts` 已有会话撤销能力，`admin.ts` 的 `revokedSessionCount` 显示此路已通）。

---

## 八、【中】方案给出的 DDL 违反本仓迁移约定

**① 哪部分**：proposal.md:9-18

**② 后果（文件:行号）**：
- 迁移文件名 `0043_audit_log.sql` 与现有序列冲突风险：`migrations/0042_audit_immutable.sql` 是当前最大版本，**0043 可用**，但方案 §1 没有写任何回滚 runbook。本仓对破坏性迁移有强制约定：需自管事务的文件**首行**必须是 `-- dsh:migration-transaction=off`（`src/server/platform/migrate.ts:31,49`，如 `migrations/0037_forum.sql:1`）；重建表必须"关 FK → 建新 → 拷 → 删旧 → 建索引 → COMMIT → FK ON"（`migrations/0039_favorite_folders_target_kind.sql:8`）。方案两种都没写。
- 表缺 `CHECK`/`NOT NULL` 约束（对比 `0001:413-425` 每列都有约束）：`action TEXT NOT NULL` 未禁止空白，`created_at INTEGER` 无 `>= 0`。

**③ 建议**：DDL 与迁移头/回滚步骤一并补齐；若采纳 §一 的建议（不建新表），本条自动消解。

---

## 九、总评与放行条件

**方案低估了问题**：仓库已有一条比方案设计**更合规**的审计链（`operation_records` + 0042 触发器 + 索引 + 幂等键 + actor 三态）。真正缺口只有三个，且都不需要新表：
1. **覆盖缺口**：审计是否覆盖全部应审计写路径（缺清单）；
2. **读取面缺口**：管理员看不到/查不方便（有 `/admin/operations` 但无导出、无"审计的审计"）；
3. **留存缺口**：审计未独立归档，且保留清理与 0042 不可变约束冲突。

**放行前必须补齐**：
- [ ] 删除 §1 新表，改为 `operation_records` 增量补齐（§一、§二）
- [ ] §3 改为 OAuth 再认证或双人复核；若坚持密码，补哈希/锁定/重置/吊销四件套（§三、§七）
- [ ] IP 一律经 `resolveClientIp`，`trustProxy` 来源与前提写入部署清单，落库做 /24、/48 收敛（§四）
- [ ] 审计独立归档 + 保留清理路径不与 0042 冲突 + `restore --force` 纳入审计（§五）
- [ ] 审计只读查询面 + 读取留痕（§六）
- [ ] 迁移头/回滚 runbook（§八）

**取证基线（可复用）**：
| 事实 | 位置 |
|---|---|
| 已有不可变审计表 | `migrations/0001_baseline.sql:411-433` |
| 已有防删改触发器 | `migrations/0042_audit_immutable.sql:17-27` |
| 审计调用点 27 处 / 11 文件 | `src/server/modules/identity/admin.ts:256,292`、`catalog/service.ts:333` 等 |
| 全站无密码（仅 OAuth） | `src/server/modules/identity/service.ts:1-16,53-56`；`src/server/platform/log.ts:24`；`src/server/modules/packs/index.ts:415` |
| IP 信任开关 | `src/server/platform/client-ip.mjs:54-71`；`docker/compose.yaml:57-59` |
| 备份仅留 14 份 | `scripts/backup.mjs:13,146,176`；`docker/compose.yaml:62` |
| restore --force 整库覆盖 | `scripts/restore-backup.mjs:97-110` |
| 限流不含鉴权类、键按用户 | `src/server/platform/rate-limit.ts:46-55,226` |
| 管理员守卫与最后管理员保护 | `src/server/platform/context.ts:46-53`；`identity/admin.ts:13-14,186-190` |
