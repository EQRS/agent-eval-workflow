# n71-admin-split SRE（运维）评审

> t54 · e3-investigator · 2026-11-17

---

# n71-admin-split SRE（运维）评审

评审对象：atb/tasks/n71-admin-split/environment/proposal.md（19 行）
已核实：src/server/platform/db.ts、health.ts、session-cookie.ts、docker/compose.yaml、
docker/nginx/conf.d/default.conf、deploy/*.service、scripts/backup.mjs、adapters/login。
并用 better-sqlite3 实测了 WAL 下双写者争抢的行为（探针已删，仓库零改动）。

## 最重要 5 条

### S1【阻塞】两个应用共用一个 SQLite 文件 = **单写者排队**，后台会把主站拖慢（已实测）
proposal:12「两个应用连同一个 SQLite 文件」。
**实测**：db.ts:53-56 是 `journal_mode = WAL` + `synchronous = FULL` + `busy_timeout = 5000`。
WAL 下**全库只有一个写者**：连接 A 持写事务时，连接 B 的 `BEGIN IMMEDIATE`
**阻塞 5.5 秒后以 SQLITE_BUSY 失败**，A 提交后 B 重试才成功。
② 后果：现在 web 与 worker 共库（dsh-review-worker.service:6 注释「SQLite 在 WAL 下支持多进程」）——
但那是「一个写者 + 一个后台任务」；再拆出一个**管理员频繁操作的后台应用**后：
  (a) 后台批量操作（导入/审核/备份）长时间持写锁 → 主站用户提交**直接 500**；
  (b) `synchronous = FULL` 每次提交都 fsync，写锁持有时间被磁盘延迟放大；
  (c) 出错形式是 SQLITE_BUSY，前端只得到 500 —— 用户看到「网站挂了」而不是「稍后再试」。
③ 建议：先明确写并发预算；拆分后(1) 后台批量写改为独立队列串行化，
(2) 后台连接用更小的 busy_timeout + 退避重试，
(3) 长期应迁 PostgreSQL，或把「后台不与主站同时跑重活」写进 runbook。

### S2【阻塞】会话 Cookie 是 `__Host-` 前缀，**无法跨域名共享**，「各自登录」会分裂账号
proposal:15「后台应用自己实现登录，复用 users 表」。
**代码事实**：session-cookie.ts:26 `SESSION_COOKIE_PROD = '__Host-dsh_session'`；
:4-6 注释「前缀要求 Secure + Path=/ + **无 Domain**」。`__Host-` 规范**禁止 Domain 属性**，
浏览器只在**单一 origin** 回传。
② 后果：`admin.example.com` 永远收不到 `review.lingshulab.com` 的会话 Cookie，
「自己实现登录」是唯一可行路径，但带来：
  (a) **两份登录态**：管理员两站各登一次；**退出只退一个，另一个仍有效**——这在共用电脑上是安全问题；
  (b) 会话撤销需两边同时生效（`account_status_version` / `management_version` 有机制），方案完全没提。
③ 建议：**不要各自登录**。要么继续单 origin 用路径区分（现状 `/admin/*`），
要么用**服务端统一会话校验**（后台每次请求回调主站 session 端点）；
若改跨子域 Cookie，需重评 CSRF/XSS 面（session-cookie.ts:4-20 记录了 `__Host-` 正是为规避那类坑）。

### S3【阻塞】迁移是**一次性独立进程**且无锁，双应用会撕裂库
**代码事实**：compose.yaml:27-33 独立 `migrate` 服务（`:4` 注释「**唯一维护进程**，web/worker 不抢」）；
web(:46-48) 与 worker(:81-83) 靠 `condition: service_completed_successfully` 等它。
而 migrate.ts **无任何锁**（grep `lock|BEGIN IMMEDIATE` 零命中）。
② 后果：两个应用各自启动时跑迁移 → 并发执行同一批 DDL → 撞车；
重则**迁移半途中断**（0007:14 已警告「中途失败会留下半成品，没有自动回滚」）。
③ 建议：迁移**收敛到唯一部署单元**（保留独立 migrate 服务，两个应用都只依赖、绝不自己跑）；
并在 migrate.ts 加显式锁（`BEGIN IMMEDIATE` + 版本表）防误用。

### S4【高】nginx/证书/回调地址全部**硬编码单一域名**
**代码事实**：default.conf:51 `server_name review.lingshulab.com`，:53-54 证书路径固定；
:9 注释已明确「**换域名**要同时改三处：server_name 与 80 的跳转地址、.env 的……」；
:42 的 301 目标也是**硬编码域名**（注释解释了为何不用 $host）。
OAuth 的 `redirect_uri` 来自环境变量（adapters/login/index.ts:37），:71-73 缺任一项即 `notConfigured`。
② 后果：新增 `admin.example.com` 需要新证书（现为单域名）、新 server 块、新 80 跳转；
若后台也走 OAuth，GitHub 上要**再加一个 callback**，否则登录直接 notConfigured。
③ 建议：方案列出完整变更清单（证书/DNS/nginx/.env/OAuth 白名单/CSP），并明确是否用通配符证书。

### S5【高】后台是**特权入口**，独立域名显著扩大攻击面
现有后台 `src/app/api/admin/**` + `src/app/admin/**` 与主站同源，由 assertCanAdmin 把守
（context.ts:46-53：userId 非空 + role==='admin' + accountStatus==='active'）。
② 后果：(a) 后台登录入口/错误页/版本号暴露在更显眼的域名上成为定向目标；
(b) 两个应用都读 users 表鉴权，**任一方鉴权 bug 等同全站沦陷**；
(c) 现有 CSRF/同源防护依赖单一 origin，跨域后要重评（方案未提）。
③ 建议：后台域名做**网络层收敛**（内网/VPN/固定 IP 白名单），nginx 加 allow/deny，不要裸暴露公网。

## 完整清单（其余）

### S6【中】备份/恢复按「一个库」设计，双应用改变恢复语义
scripts/backup.mjs:6-8 用 SQLite 在线热备 `db.backup()`（不阻塞写入，这点很好），:65 之后假设“单一写入来源”。
② 后果：恢复时要同时停两个应用，否则出现半个库被覆盖；`restore-backup.mjs --force` 风险面扩大。
③ 建议：runbook 写死「恢复必须先停 web + worker + admin 三个单元」，做成脚本前置检查。

### S7【中】健康检查断言 WAL，无法区分“谁把库锁住了”
health.ts:73 `ok: info.journalMode === 'wal'`。
② 后果：S1 的 SQLITE_BUSY 不会让健康检查失败（库仍是 WAL、连得上）→
**编排器显示 healthy 但用户全 500**，最难查的一类故障。
③ 建议：暴露「最近 N 秒 SQLITE_BUSY 次数」指标并纳入告警。

### S8【中】部署单元翻倍，还共用同一份 .env.local
deploy/ 现有 web/worker 两个单元，均 `WorkingDirectory=/srv/dsh-plugin-review` + 同一 `.env.local`。
② 后果：新增第三单元后共享 env，改一个变量同时影响两边；
独立构建意味着两套 node_modules（磁盘/内存翻倍，低配 VPS 不可忽视）。
③ 建议：独立 `.env.admin`；方案给出资源预算（内存/磁盘/构建时间）。

### S9【低】没有回滚路径
proposal:17「不影响主站」——但**拆分本身**就影响主站（S1 写锁、S2 会话、S3 迁移）。
建议明确灰度方式（先只读后台？）、回滚步骤（DNS 切回 + 恢复单一部署）与回滚窗口。

### S10【低】「不影响主站」缺可验证判据
建议量化：拆分前后主站 P99 延迟、SQLITE_BUSY 次数、部署期间 5xx 率。

## 一句结论
拆分方向可以理解，但方案只有 3 行措施，却踩中本仓运维**最脆弱的三处**：
**SQLite 单写者**（实测双写者 busy 5.5s 后失败，S1）、
**`__Host-` 会话 Cookie 不可跨域**（S2，必然导致两份登录态与退出不同步）、
**迁移是唯一维护进程且无锁**（S3，双应用并发迁移会撕裂库）。
建议：**优先不要拆域**；若必须拆，先解决写并发预算、统一会话、迁移收敛，
并给出证书/DNS/OAuth/回滚的完整变更清单。