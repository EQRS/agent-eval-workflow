## 安全工程师评审：tasks/n71-admin-split/environment/proposal.md

### 摘要（前 5 条最重要）

1. 【严重/阻断】方案 §3「后台应用自己实现登录，复用 users 表」会造出第二套认证实现，而与现有实现等价的第二套在安全上做不到。依据：`src/server/modules/identity/service.ts:245-276`（resolveSession：token_digest 查库 + revoked_at/expires_at/absolute_expires_at 三重判断 + 每次从库读 role/account_status）、`:93-114`（bootstrapAdminWithin 只在「零可用管理员」时提升，防降权后复活）、`src/server/modules/identity/oauth-challenges.ts:60-78`（return_path 开放重定向防护）、`:110-117`（生产无 SESSION_SECRET 拒绝发起登录）、`src/server/platform/context.ts:46-53`（assertCanAdmin）、`src/server/modules/identity/admin.ts:237-245`（服务层自查 actor 仍在任，「绝不提权」）。这六条不变量任漏一条，后台即为提权通道。建议：删除「自己实现登录」，改为同一个 identity 模块 + 同一个 Cookie + 同一套 users.role；若必须双应用，认证必须是同一份代码的共享库（抽为内部包），并配「管理员断言在服务层」的回归用例。

2. 【严重】§2「两个应用连同一个 SQLite 文件」破坏本仓「web 与 worker 共用库 = 信任边界已在库内」的前提，却没有改判任何安全假设。依据：`src/server/platform/db.ts:44-64`（每进程自连、PRAGMA 各自设定）；审计不可变只有触发器兜底 `migrations/0042_audit_immutable.sql:17-27`（BEFORE DELETE/UPDATE），而应用层 `src/app/api/admin/backups/route.ts:110-119` 会 fork `scripts/backup.mjs`、`scripts/restore-backup.mjs:98-111` 支持 --force 整库覆盖并清理 -wal/-shm，`src/server/platform/migrate.ts:117-119` 的迁移在任一连接上都能执行。结论：拆分后主站侧任何进程都能改写后台的权限基线；一旦备份产物落入另一应用的写权限范围，0042 的「审计不可变」就形同虚设（该迁移注释自述「删除审计行 + restore --force 构成无需改代码的伪造通道」）。建议：拆分不得让两应用都持可写句柄——后台写、主站只读（`:23` 已支持 readonly 连接），或按表拆库。

3. 【严重】跨站 Cookie 与 CSRF 面在方案里完全未提。依据：现会话 Cookie `src/server/platform/session-cookie.ts:92-101` 为 __Host- + Secure + SameSite=Lax + Path=/，而 __Host- 前缀要求 Path=/ 且无 Domain，无法限制到 admin 子域。两种简化都危险：(a) 写成 .example.com 共享域 → __Host- 前缀失效，主站任意 XSS 或子域接管即可窃取后台会话；(b) 两应用各发 Cookie → 两套会话并存，「撤销/禁用立即生效」（`identity/service.ts:262-267`）要两边都实现。另：全仓无任何 Origin/Sec-Fetch-Site 校验，唯一防线是 SameSite=Lax，而 Lax 不防顶级表单 POST。建议方案明确：Cookie 不跨域共享、后台用独立 __Host- Cookie、所有写接口校验 Origin/Referer + Sec-Fetch-Site。

4. 【高】nginx 安全基线是按「单入口」写死的，拆出 admin.example.com 后不会自动跟随。依据：`docker/nginx/snippets/proxy-common.conf:15,23` 明确 X-Real-IP/XFF 用 $remote_addr 覆盖，:20-22 警告「再加 CDN/负载均衡必须改 set_real_ip_from + real_ip_header」；`docker/nginx/conf.d/default.conf:41-42` 把 80 全部 301 到硬编码 review.lingshulab.com；:59 的 HSTS 只加在 443 站；`site-locations.conf:16-18` 靠 nginx 兜底 deny /api/testing/session。新站点若复用同一片上联或另起反代，客户端伪造 X-Real-IP 即重新生效，日志/限流/审计的真实 IP 全部失真。另：后台暴露公网后，`src/app/api/health/route.ts:25-27` 的 x-health-token 是明文 === 比较（非常量时间），新增第二个公网面即扩大探测面。

5. 【高】§3「复用 users 表」语义是错的，且放大越权面。依据：`src/server/modules/roles/service.ts:12-18` 明确 users.role（user/admin）决定能否进后台，user_roles 只是能力标记；后台真正吃的是 users.role='admin' AND account_status='active'（`identity/admin.ts:187-193`、`context.ts:23-25`）。方案说「复用 users 表」却没说要复用 users.role 与最后一名管理员守卫（`identity/admin.ts:13-16,329`），含义是再写一套角色判定；两套一旦漂移，就会出现「主站可写、后台可审」的身份错配。建议把权限口径写成可验收断言：后台准入 = users.role='admin' AND account_status='active'，且该判定只允许存在一份实现。

### 其余发现

6. 【中】拆分把审计与幂等载体暴露给更多写入方。operation_records 的 UNIQUE(action_type, operation_key) 既是审计也是幂等回执（`identity/admin.ts:23-24,253-274`）；后台拆出后若「自己实现审批流」，幂等键与 request_digest 构造规则必须共享，否则同一次管理操作可被两应用各执行一遍。方案「预期」里的「后台独立迭代」正是这条规则漂移的最大来源。

7. 【中】「不影响主站」的预期与安全事实相反。两应用同时改 .data/app.sqlite 与 media（`docker/compose.yaml:64-67` 已分别挂载数据/备份/媒体），而 SQLite 是 WAL 单写者（`db.ts:53-56` synchronous=FULL）。任一应用执行迁移（`migrate.ts`）或恢复（`restore-backup.mjs`）都是跨应用的结构性变更。建议方案显式写出既有约束：破坏性操作只允许由 migrate 一次性容器执行（`compose.yaml:26-33`）。

8. 【中】凭据面翻倍。`docker/web.Dockerfile:67` 规定秘密一律运行时注入、绝不烘进镜像，而 `compose.yaml:34/49/84` 三处共享同一份 env_file: .env——即 SESSION_SECRET、GITHUB_CLIENT_SECRET、GITHUB_TOKEN、BOOTSTRAP_ADMIN_GITHUB_ID 同处一文件。拆两应用若各拿一份 .env，凭据复制面与轮换成本翻倍；且 BOOTSTRAP_ADMIN_GITHUB_ID（`identity/service.ts:82-114`）若配到后台应用，会引入第二个「引导提权」入口。建议规定后台应用不持有 GITHUB_TOKEN / BOOTSTRAP_ADMIN_GITHUB_ID，OAuth 只由主站一处发起。

9. 【中】「共享数据库」只写一句，缺三条安全前提：①库文件权限（现靠容器 UID 10001 对齐，`compose.yaml:20-21`，宿主属主不同即失败）；②若后台应用部署在另一台机器，SQLite 共享不可行（WAL 不支持网络文件系统），方案未标「必须同机同卷」；③网络暴露——现 web 刻意不发布端口（`compose.yaml:63`），拆分往往导致各自发端口，需重申「仍然只有 nginx 一个入口」。

10. 【低】方案无数据分级表述，而后台访问内容明显更高敏：`src/app/api/admin/users/route.ts:16` 的列表已含全部用户 GitHub 登录名与禁用状态；备份接口等价于「一键导出全站数据」。拆分后后台成为独立公网高价值目标，方案至少应增加：后台入口独立限流、管理员 MFA 或二次确认、后台访问日志单独留存（0042 保护的是库内审计，不含 nginx 日志）。

### 结论
按方案字面实施（§3 尤其）不建议放行：它把「重新实现认证」当成平凡步骤，而本仓认证不变量是可枚举、可漏掉的六条。可行方向是「同一 identity 代码 + 后台只读库句柄 + 独立 __Host- Cookie + 独立主机名但同一入口」，这些必须在方案里显式落字，而不是留给实施者各自决定。
