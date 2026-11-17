# 评审：n71-admin-split「管理后台改造成独立应用」方案

评审对象：`tasks/n71-admin-split/environment/proposal.md`（19 行，全文仅 3 个技术要点 + 2 条预期）
结论：**不予通过。** 方案本身没有写错什么，但也没写对任何东西——它把本仓三个最危险的耦合面（单一 SQLite 文件、唯一鉴权入口、唯一入口 nginx）当成"共享数据库""自己实现登录"一句话带过，而这三处恰好都在仓库里有明确的、被真实事故验证过的约束。

---

## P0（阻断级：照此实施会直接造成安全或数据事故）

### 1. ①「共享数据库」= 两个应用连同一个 SQLite 文件，且与 WAL 热备共用一份数据目录
- **方案位置**：proposal.md:11-12「两个应用连同一个 SQLite 文件」。
- **后果**：
  - 本仓的并发模型是**进程级单例 + 单文件 WAL + `busy_timeout=5000` + `synchronous=FULL`**（`src/server/platform/db.ts:44-64`、`:66-74`）。当前已经是 web 与 worker **两个进程**共享一个库（`deploy/dsh-review-web.service:19`、`deploy/dsh-review-worker.service:17`、`docker/compose.yaml:39,54,89`），拆出后台后变成**第三个常驻写进程**，且写入频率集中（审核、批量导入、备份）。
  - better-sqlite3 是**同步驱动**，等待会阻塞所属线程（`db.ts:7-8` 自己写着这条）。第三个写进程把写锁占用拉长后，主站的写请求会在 `busy_timeout` 上限后被判 `SQLITE_BUSY`；这不是"慢一点"，是**主站用户提交/评论/点赞随机失败**——而方案的"预期"恰恰承诺"不影响主站"（proposal.md:19）。
  - 更危险的是**备份与媒体的路径耦合**：备份用在线热备 `db.backup()` 并同时打包媒体目录（`scripts/backup.mjs:6-12,39-43`），后台的"立即备份"按钮直接 `execFileSync` 这个脚本、`cwd=process.cwd()`、`env: {...process.env}`（`src/app/api/admin/backups/route.ts:110-119`）。拆成两个应用后，**两个进程各有各的 `process.cwd()` 与 `DSH_REVIEW_DB_PATH` 默认值**（`db.ts:33` 默认 `'.data/app.sqlite'` 是**相对路径**，按 cwd 解析、按需 mkdir）。只要后台容器的 `WorkingDirectory` 或 `DSH_REVIEW_DB_PATH` 有一处没对齐，后台的"备份"就会对着**自己新建的空库**跑，产出一份 structure-valid 但内容为空的备份，覆盖保留窗口里的真实备份。这类事故没有任何报错，只有恢复时才发现。
  - 单文件 SQLite **不允许跨主机/跨挂载点共享**。如果"独立应用"最终部署到另一台机器（方案没排除），或者数据目录通过网络盘挂载，WAL 的一致性假设直接失效，损坏是概率事件而非异常事件。
- **建议**：
  - 方案里必须**显式钉死**：两个应用是否同宿主、是否同一进程用户（当前固定 `10001:10001`，见 `docker/compose.yaml:20-21`）、数据目录的**绝对路径**分别在哪个配置文件里定义（不要留相对路径默认值），以及迁移（`migrate` 一次性服务，`compose.yaml:27-42`）由谁独占执行——拆成两个应用后**必须仍然只有一个迁移执行者**，否则两边的 schema 版本会打架。
  - 给出写压力量化：后台改造后新增的写入 QPS 估算，与 `busy_timeout=5000` 的余量对比；拿不出数就说明这份方案没有评估过"不影响主站"这个承诺。
  - 备份入口必须**只保留一个**（建议留在后台，由后台调用一个带**绝对库路径**的备份 CLI，且不接受 cwd 派生）；同时明确"备份是全局资源，不能两边各备一份"。

### 2. ①「后台应用自己实现登录，复用 users 表」
- **方案位置**：proposal.md:14-15。
- **后果**：
  - 本仓的鉴权**不是**"users 表里有密码"这种可复用资产：会话是 `__Host-dsh_session` Cookie（`src/server/platform/session-cookie.ts:26-28`），库里只存摘要，解析走 `identity.resolveSession` 且**每次请求从库读角色**（`src/app/api/_lib/context.ts:21-28`、`src/server/platform/context.ts:1-5`）。没有密码列、没有独立登录态。"自己实现登录"等于**从零造第二套凭据体系**，却仍要读同一个 `users` 表。
  - 直接后果是出现**两套撤销语义**：现在禁用账户会在同一事务里撤销该用户全部会话（`src/server/modules/identity/admin.ts:565-572`），且服务层自己再校验一次 actor 是不是在任管理员（`admin.ts:236-245`，注释明写"自我提权连服务层都进不去"）。后台若自建登录态，`admin.ts:569` 那条 `UPDATE user_sessions SET revoked_at` **撤销不到后台的会话**——被禁用/被撤销管理员的人，后台仍然在线。这是最典型的权限回收失效。
  - 若改成"后台发新 Cookie"，由于主站用的是 `__Host-` 前缀（`session-cookie.ts:26`，要求 Secure + Path=/ + **无 Domain**），后台在 `admin.example.com` 下**无法复用**同名 Cookie；而若退化成 `Domain=.example.com` 共享，就破掉了 `__Host-` 的全部保护，主站的会话凭证会暴露给后台子域的任何 XSS。
  - 方案也没提 GitHub OAuth 的绑定 cookie 与 PKCE 挑战（`src/app/api/auth/github/callback/route.ts:56-64`、`src/server/adapters/login/index.ts:34-40`）：`GITHUB_REDIRECT_URI` 是单值环境变量，换域名必须同时改 redirect_uri 与 GitHub App 回调地址——`docker/nginx/conf.d/default.conf:9-10` 已经把"换域名要同时改三处"写成警告，方案对此只字未提。
- **建议**：
  - 后台**不要自建登录**。明确二选一并写进方案：(a) 后台复用同一 `/api/auth/github/*` 与同一会话解析，只是域名不同（此时必须解决 `__Host-` 不能跨子域的硬约束，通常意味着**不换域名**而是换路径前缀 `/admin`）；(b) 若必须独立域名，则要给出 SSO 方案（短时票据 + 后台侧每请求回源校验角色 + 后台会话纳入同一套 `user_sessions` 撤销链路），并明确后台会话**必须**受 `admin.ts` 的撤销逻辑覆盖。
  - 明确"禁用/撤销管理员后多长时间内后台必须踢下线"，并把这个时限写成可测的验收项。

### 3. ①「路由迁到 admin.example.com」——入口与真实 IP 信任模型
- **方案位置**：proposal.md:9。
- **后果**：
  - 本部署的安全前提是 **nginx 是唯一对外入口**，因此应用可以安全信任 `X-Real-IP / X-Forwarded-For`（`docker/nginx/conf.d/default.conf:1-5`、`docker/nginx/snippets/proxy-common.conf:13-23`）。compose 里 web **刻意不发布端口**（`compose.yaml:63`），并把 `DSH_REVIEW_TRUST_PROXY=1` 与"若直接发布端口必须改成 0"写死成注释（`compose.yaml:57-59`）。
  - 拆出新应用 = 新增一个对外入口。方案没有说明新入口是否也过同一 nginx、是否共用 `proxy-common.conf`、以及后台服务自身的 `TRUST_PROXY` 取值。**只要后台沿用 `TRUST_PROXY=1` 却被直接暴露端口/被另一层反代进去，任何人都能伪造 `X-Real-IP`**，而 `X-Real-IP` 正是频控与审计的记录依据。
  - `proxy-common.conf:16-22` 明确警告"将来若在前面再加 CDN/负载均衡，必须改成 `set_real_ip_from` + 追加语义"——"admin 独立域名"几乎必然意味着新的代理层，而方案没有触碰这条约束。
  - 频控存储也在同一个库里（`migrations/0006_rate_limiter.sql:25-32`，表 `rate_limiter_flexible`）。两个应用若各自一套 limiter 且共享同一张表，键空间会互相覆盖；若不共享，则"同一 IP 对主站的攻击"不会反映到后台的阈值上——审计与防护口径分裂。
- **建议**：画出新拓扑（谁是对外唯一入口、IP 头由谁写、`TRUST_PROXY` 两边分别取什么值），并逐条对照 `proxy-common.conf` 的注释检查是否踩坑；频控键空间必须显式规定（共享 / 隔离 / 加前缀）。

---

## P1（严重：方案缺项，实施中必然返工）

### 4. ①拆分边界完全没定义——实际耦合面远大于"后台路由"
- **方案位置**：proposal.md:3-9 只有一句"后台功能越来越多"。
- **后果**：仓里 `assertCanAdmin` 出现在**28 个 admin API 路由**里（例：`src/app/api/admin/settings/route.ts:12,23,38`、`src/app/api/admin/backups/route.ts:17,81,104`、`src/app/api/admin/imports/[batchId]/run/route.ts:12,26`），并且**管理逻辑大量沉在 `src/server/modules/` 与主站共用**（`src/server/modules/content/index.ts:14,51`、`src/server/modules/forum/applications.ts:17,167,183,242`、`src/server/modules/forum/boards.ts:14,348,425`）。另有至少一个**主站侧**路由直接复用管理员权限：`src/app/api/pack-releases/[releaseId]/review/route.ts:5,18`。后台页面另引用 20 处 `@/components/*` 共享组件。
- **建议**：先出一份"拆分清单"：哪些模块进后台、哪些留在主站、"后台路由"按 `src/app/api/admin/**` 还是按语义划分。没有这份清单，"独立构建"无法估工作量，也无法判断是否会把 `pack-releases/[releaseId]/review` 这种混合入口撕裂。

### 5. ①"复用 users 表"隐含双写权限，但没有写权限边界
- **方案位置**：proposal.md:12,15。
- **后果**：后台独立后仍是**同一个库的超集权限**。现在"至少保留一名可用管理员"的守卫是在**同一事务内**数 `users` 表并与 UPDATE 原子化（`src/server/modules/identity/admin.ts:401-417`、`:532-548`）。如果后台与主站被实现为两个应用，任何一个新写的写路径（导入、批量、运维脚本）都可能绕过这条守卫。而 `scripts/grant-admin.mjs` 的存在说明历史上确实有人直接 `UPDATE users SET role='admin'`（`admin.ts:5-9` 注释）。
- **建议**：方案必须声明"后台是否允许 `grant-admin` 类旁路"，以及**断言的唯一性**：所有 `users` 变更必须走 `identity/admin.ts` 的那两个函数。建议把这条写成 CI 门禁（禁止新的直接 `UPDATE users`）。

### 6. ①没有回滚与灰度策略
- **方案位置**：proposal.md:17-19「预期」只有两条好处，没有失败路径。
- **后果**：本仓对"可回退"是有强约定的（`docs/备份与恢复.md`、`scripts/restore-backup.mjs` 需在维护窗口执行，见 `src/components/admin-backup-panel.tsx:7,87`）。拆分后台属于**跨两个应用的结构性变更**，方案没有写：迁移是否可逆、切换是否双跑、主站后台入口是否保留一段时间、切坏的判定标准是什么。
- **建议**：补一节"回滚方案"：切换前 `node scripts/backup.mjs --label pre-deploy`；保留旧 `/admin` 路径 N 天；给出"后台判定不可用"的具体标准（心跳/错误率/审计连续性）与 5 分钟内的回切步骤。

### 7. ①可观测性与审计连续性没写
- **方案位置**：通篇未提（proposal.md:1-19）。
- **后果**：现在 web 与 worker 的日志靠 `SyslogIdentifier` 区分、worker 有独立心跳健康检查（`deploy/dsh-review-worker.service:25`、`compose.yaml:102-108`）。新增一个应用就多一份日志、一个心跳、一条 `X-Request-Id` 传递链（`proxy-common.conf:30-32`）。而审计表 `operation_records` 是**不可变**的（`migrations/0042_audit_immutable.sql`），跨应用写入时 `request_key` 幂等键空间必须**全局唯一**——目前幂等是 `(action_type, operation_key)` 唯一（`admin.ts:23-24`）。两个应用若各自生成 `request_key`，碰撞会表现为 `request_key_reused` 409 或**误判为同一次操作的静默重放**。
- **建议**：规定 `request_key` 的生成规则（建议带应用前缀），规定 `X-Request-Id` 必须透传到后台并落审计；后台必须有按 worker 同款方式的心跳健康检查。

---

## P2（建议补充）

### 8. ①环境变量与配置漂移
- **后果**：`.env.example`、`docker/.env.example` 是单一来源；拆出第二个应用后必然出现两份 `env`，而当前有多个**必须一致**的变量：`DSH_REVIEW_DB_PATH`（`compose.yaml:39,54,89`）、`SESSION_SECRET`、`TZ=Asia/Shanghai`（`compose.yaml:38,52,88`，直接影响备份目录命名，`scripts/backup.mjs:51-53`）、`BACKUP_ROOT/BACKUP_KEEP`（`compose.yaml:61-62`）。`TZ` 两边不一致会让备份目录名与日志时间错位。
- **建议**：把"必须两边一致"的变量列成清单并加自检（启动时比对 `schema_version` 与关键 env），不一致直接拒启——与本仓"证书不在位就启动失败"的既有取舍（`docker/nginx/conf.d/default.conf:11-12`）保持一致。

### 9. ①方案颗粒度不达可评审标准
- **后果**：19 行里没有任何一条可验证的验收项。"后台独立迭代""不影响主站"（proposal.md:18-19）都不是可测断言，无法写进测试或门禁。
- **建议**：至少补：拆分后 `/admin` 与 `admin.example.com` 的行为等价范围、主站 P95 延迟变化上限、迁移时长、切换停机窗口、回滚时间上限。参考本仓既有任务都带 `tests/criteria.md` 的惯例（`tasks/n71-admin-split/tests/criteria.md` 已存在），方案必须能映射到那些判据上。

---

## 总评
这份方案的问题不在于"写得不好"，而在于**它把一个架构决策（单体拆分）当成了一次配置搬家**。本仓有三个被反复强调、且都有代码/注释/事故记录支撑的硬约束：**单文件 SQLite + 同步驱动 + 多进程**（`db.ts:7-8,44-64`）、**唯一鉴权入口 + 每请求读角色 + 统一撤销链路**（`context.ts:21-28`、`identity/admin.ts:565-572`）、**nginx 唯一入口 + 可信 IP 头**（`conf.d/default.conf:1-5`）。方案对这三条一条都没有回应，且第 11-15 行恰好同时踩在第二条上。

最小可接受修订：把第 2、3 节展开成"部署拓扑图 + 鉴权方案对比（含 `__Host-` 约束）+ 迁移唯一执行者 + 数据目录绝对路径 + 回滚步骤 + 至少 3 条可测验收项"。在补齐之前，建议**不进入实施**。
