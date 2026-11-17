# SRE（运维）评审：n71-admin-split「管理后台改造成独立应用」

评审对象：`atb/tasks/n71-admin-split/environment/proposal.md`（19 行）
基线：本仓 2026-11 现状（Next 16 + better-sqlite3 + SQLite WAL/synchronous=FULL/busy_timeout=5000；compose 四角色 migrate/web/worker/nginx；`deploy/` 下 web/worker 双 systemd 单元）
视角：**只从 SRE（运维）角度**——值守、备份恢复、发布回滚、容量成本、可观测性、失效可见性。
结论：**不予通过。** 方案 3 行措施里，「共享 SQLite」与「自建登录」直接落在本仓运维最脆弱的三处上；更关键的是方案通篇没有运维面（无备份/恢复、无回滚、无容量、无监控口径），并承诺了一个它无法证明的「不影响主站」。

---

## S-1【P0】「不影响主站」无运维证据，且现行部署没有任何容量/故障信号

**① 问题**
方案 `proposal.md:19`「不影响主站」，全文 19 行无任何量化。

**② 后果**
- 现行唯一跨进程存活信号是 worker 心跳（`src/server/platform/heartbeat.ts:17-23`、`docker/compose.yaml:102-108`），它只回答「worker 还活着」；web 侧 500 率、`SQLITE_BUSY` 次数、P95 都没有采集点（`/api/health` 只做自检：`src/server/platform/health.ts:56-147`）。
- 拆出第三个常驻写进程后，「主站变慢/提交失败」只能是**用户报障**才被发现——运维上这是最贵的一类：没有可观测入口，就没有运维。
- 回滚判据不存在，等于「切过去之后出事，无法判断该不该切回」。

**③ 建议**
实施前先补三个数：主站写请求 QPS、P95 提交延迟、`SQLITE_BUSY`/5xx 率基线（1 周窗口）。拆分方案必须带「主站 P95 增幅 ≤ X%、busy 率 ≤ Y%」的硬判据，并先把 busy 计数打在 `server.mjs:76-81` 已有的每请求日志旁边（不必新起服务）。

---

## S-2【P0】双写进程让「同步等待 + busy_timeout」从偶发变成常态故障，且故障形态没有分类

**① 问题**
`proposal.md:12`「两个应用连同一个 SQLite 文件」。

**② 后果**
- `src/server/platform/db.ts:52-56`：`journal_mode=WAL` + `synchronous=FULL` + `busy_timeout=5000`；`db.ts:7-8` 自述「better-sqlite3 是同步驱动，等待会阻塞所属线程」。
- 现状已是 web + worker 两进程共库（`docker/compose.yaml:39,54,89`、`deploy/dsh-review-worker.service:6` 注释「SQLite 在 WAL 下支持多进程」）。拆出后台 = **第三个常驻写进程**，而它的写是管理员批量操作（导入/审核/批量改），持锁时长远超 worker 的单条任务。
- 后果不是「慢」：`busy_timeout` 到期抛 `SQLITE_BUSY`，冒泡成 500（`src/server/platform/errors-http.ts` 无 SQLITE_BUSY 专项映射），用户看到「网站挂了」而不是「稍后再试」；且 5 秒同步等待期间该进程**整条事件循环停摆**，主站所有请求一起卡。
- 最难查的形态：进程健康、库是 WAL、`/api/health` 返回 ok，但用户全 500（`health.ts:73` 只断 `journalMode === 'wal'`）。

**③ 建议**
(1) 方案写死写并发预算：后台批量写必须走**串行队列**（复用 `background_tasks` 通道），不在请求线程里做大批量写；
(2) 后台连接用更小 `busyTimeoutMs` + 退避重试，主站连接保持 5000；
(3) `errors-http.ts` 给 SQLITE_BUSY 一个 503 + `Retry-After` 的专门映射；
(4) `health.ts` 增加「最近 N 秒 busy 次数」项，超阈值判 degraded 并进告警。

---

## S-3【P0】迁移没有互斥锁，双应用各自启动会撕裂库；`0007` 无事务分支还会留半成品

**① 问题**
`proposal.md:11-12` 隐含两个应用各自连库、各自启动；方案未提「谁执行迁移」。

**② 后果**
- `docker/compose.yaml:1-8,25-33` 把「migrate 是**唯一维护进程**，web/worker 不抢」写成部署契约；两个应用若各自跑 `migrate()`（`src/server/platform/migrate-cli.ts:4`），`migrate.ts:88-123` **没有任何锁**（无 `BEGIN IMMEDIATE`、无文件锁），并发 DDL 直接撞车。
- `migrate.ts:97-115` 的 `-- dsh:migration-transaction=off` 分支（现只有 `0007`）**不自动回滚**，中途失败留半成品，下次启动因「表已存在」必炸，只能人工补登。
- 两应用版本不同步时，先起的一方执行新迁移，后起的一方用旧代码读新 schema，错误形态是运行期列缺失。

**③ 建议**
迁移**保持唯一执行者**：独立一次性服务（现状 `migrate`），两个应用只 `depends_on: service_completed_successfully`，任何应用**禁止**自己调 migrate；并在 `migrate` 里加互斥（`BEGIN IMMEDIATE` 或文件锁），把「误用」变成响亮的失败。方案须写明「schema 前进后旧应用的行为」。

---

## S-4【P0】备份/恢复契约会静默失效——「以为有备份」是最贵的一类运维事故

**① 问题**
方案未提备份；而备份与「决定只有一个应用」强绑定。

**② 后果**（三处独立的坑，任一处即产出「看起来成功、实际不可用」的恢复点）
- **库路径相对值**：`src/server/platform/db.ts:33` 默认 `'.data/app.sqlite'` 是**相对路径**，按 `process.cwd()` 解析并 `mkdirSync`（`db.ts:35-37`）；`scripts/backup.mjs:39`、`scripts/restore-backup.mjs:30` 同款。后台容器的 WorkingDirectory 或 `DSH_REVIEW_DB_PATH` 只要差一处，后台「立即备份」（`src/app/api/admin/backups/route.ts:110-119`，`cwd=process.cwd()`）就对着自己新建的空库备份：manifest 通过 `integrity_check`、退出码 0。
- **保留策略互踩**：`scripts/backup.mjs:146-157` 按目录名排序只留 `--keep` 份（compose 默认 `BACKUP_KEEP=14`，`docker/compose.yaml:62`）。两个应用各自定时备份同一个 `BACKUP_ROOT`（`compose.yaml:60-62,67`）会**各删一半**对方的恢复点，实际保留窗口塌成 ~7 天且无人察觉。
- **恢复无人值守**：`docs/备份与恢复.md:66-75` 的恢复流程只停 `web worker`。被打开的 SQLite 文件**不能替换**（`db.ts:66-74` 进程内单例持连接）；漏停后台进程时，恢复写入的新库被旧进程的 WAL/长连接覆盖，恢复「成功」但数据仍是旧的。`scripts/restore-backup.mjs:108-110` 删 `-wal/-shm` 只对已停止的进程安全。

**③ 建议**
(1) 库与备份路径在两边配置里写成**绝对路径**，并在 `backup.mjs` 里断言 dbPath 存在且关键表行数 > 0（现在是「文件存在就继续」，空库也过）；
(2) **备份入口只保留一个**（建议后台调用带绝对库路径的 CLI），另一侧物理上不给 `BACKUP_ROOT` 写权限；
(3) 恢复 runbook 与 `restore-backup.mjs` 前置检查改为「停 web + worker + admin」，脚本自身校验目标库无进程占用。

---

## S-5【P1】健康检查与存活信号不覆盖新单元，编排器会「显示 healthy 而站点已死」

**① 问题**
方案未提新应用的健康检查、心跳、重启策略。

**② 后果**
- 现只有 worker 有心跳健康检查（`compose.yaml:102-108` + `scripts/worker-healthcheck.mjs`）；web 靠 nginx `depends_on: service_healthy`（`compose.yaml:115-118`）。新增单元若无等价健康检查，编排器只会看「进程在」，`restart: unless-stopped` 会对着一个连不上库/被锁死的进程**无休止重启**并放大锁竞争。
- `/api/health` 的 `statusForHealth` 只有 fail 才 503（`health.ts:149-152`），degraded 仍 200——若新单元只接这个端点，busy 风暴不会触发任何重启或告警。
- 后台是特权面：它挂了不等于站点挂了，但它**沉默地挂着**（管理员操作全部失败）同样要被编排发现。

**③ 建议**
后台补一条与 worker 同级的心跳（复用 `runtime_heartbeats`，`src/server/platform/heartbeat.ts:132-151`）+ `HEALTHCHECK`；明确「后台不可用」的判定口径（心跳 stale / health 非 ok / 5xx 率）与告警路径。

---

## S-6【P1】发布与回滚路径未定义，拆域把「换域名」变成一次多面同改的高危操作

**① 问题**
`proposal.md:9`「后台路由迁到 admin.example.com」；方案全文无发布/回滚步骤。

**② 后果**
- 域名与证书是**硬编码三处同改**：`docker/nginx/conf.d/default.conf:9-12` 明写「换域名要同时改 server_name 与 80 跳转地址、`.env` 的 GITHUB_REDIRECT_URI/SITE_ORIGIN、GitHub OAuth App 的 callback」；`:42` 的 `return 301 https://review.lingshulab.com$request_uri` 写死，`:51` 的 `server_name review.lingshulab.com` 是单域名。证书只挂一个目录（`compose.yaml:123-125`），**新域名要么再签一张要么换通配符**。
- `deploy/renew-hook.sh:14-16` 的续期钩子写死 `DOMAIN`/`DEST`/`COMPOSE_DIR`，`:27-30` 只 reload nginx——新增域名后续期链路**要么漏签要么漏 reload**，到期当天全站证书报错（该文件 `:5-6` 注释已写明这是历史风险）。
- 无回滚：一旦切过去出问题，切回需要「DNS 回退 + nginx 回退 + 证书回退 + OAuth 白名单回退」四步，方案没给窗口与判定标准。

**③ 建议**
补一节「发布与回滚」：切换前 `node scripts/backup.mjs --label pre-deploy`；域名/证书/OAuth 的完整变更清单（含通配符证书决策与 `renew-hook.sh` 改造）；保留旧 `/admin` 路径 N 天；给出「5 分钟内回切」的具体命令序列与验证点。

---

## S-7【P1】日志与排障链路分裂，事故时间线拼不起来

**① 问题**
方案只谈「独立构建/部署」，不谈日志与追踪。

**② 后果**
- 现行日志三处独立：应用每请求 JSON 行（`server.mjs:39-43,62-82`，含 `request_id`/`client_ip`/`status`/`duration_ms`）、nginx `access_log`（`docker/nginx/nginx.conf:29`，格式见 `:24-27`）、systemd journal 按 `SyslogIdentifier` 分单元（`deploy/dsh-review-web.service:31`、`worker.service:25`）。容器侧 json-file 10m×3（`compose.yaml:70-74,95-99,127-131`、`docker/daemon.json:3-7`）。
- 新增应用 = 第四份日志 + 一条新的 `X-Request-Id` 传递链（`docker/nginx/snippets/proxy-common.conf:30-32`）。若后台不沿用同一注入约定，「用户报一个编号 → grep 到整条链路」的既有排障能力在跨应用场景**直接失效**。
- 日志保留 10m×3，一次 busy 风暴会把事故前的窗口冲掉。

**③ 建议**
写明：新入口必须走同一 nginx 且透传 `X-Request-Id`（`proxy-common.conf:30-32` 已有约定，不能另起一套）；日志落同一点、带单元前缀；审计/错误级别单独延长保留（10m×3 不够事故取证）。

---

## S-8【P1】真实 IP 信任模型与限流键空间在多入口下失控

**① 问题**
`proposal.md:9` 新增对外入口，但未定义 IP 头由谁写、限流键怎么算。

**② 后果**
- `server.mjs:30-32`：`DSH_REVIEW_HOST` 默认 `0.0.0.0`、`TRUST_PROXY` 默认关；`compose.yaml:57-59` 与 `docker/.env.example:50-52` 都把 `DSH_REVIEW_TRUST_PROXY=1` 连同「本端口不得直接暴露公网」写死。后台只要沿用 1 又被直接暴露（或前面多挂一层未按 `proxy-common.conf:20-22` 改 `set_real_ip_from` 的代理），客户端即可伪造 `X-Real-IP`——而这正是频控与审计的记录依据。
- **限流额度互相抵扣**：`rate_limiter_flexible` 的键是 `${action}:${userId}`（`src/server/platform/rate-limit.ts:226`，表结构 `migrations/0006_rate_limiter.sql:25-32`），**不含应用维度**。两个应用共享同一张表与键空间 = 同一管理员在两个入口共用一份额度（不是翻倍，是各自被对方扣掉）；不共享 = 两套口径。另：30 个 admin 路由**完全没有频控**（`enforceRateLimit` 26 处调用点全在用户侧），管理员在两个入口重复点「立即备份」只会撞 `backups/route.ts:74-75,105` 的 `running` 标志——**该标志是进程内变量**，双进程下形同不存在。

**③ 建议**
画出新拓扑并钉死：唯一入口、`TRUST_PROXY` 两边取值、后台是否网络层收敛（nginx `allow/deny` 或内网）。限流键加应用前缀或独立表；`running` 这类进程内互斥改成库内互斥（唯一键/租约行），否则跨进程无效。

---

## S-9【P2】容量与磁盘：独立构建 = 资源翻倍，且没有资源预算

**① 问题**
方案未给资源评估。

**② 后果**
- 独立构建 + 独立 `node_modules`（`deploy/dsh-review-worker.service:16-17` 已因 tsx 而必须装 devDependencies）、独立容器、独立日志轮转；低配 VPS 上内存/磁盘是硬约束。
- 磁盘余量只有 200MB 阈值一个信号（`health.ts:50,88-103`）；备份 14 份 + 媒体 + 双份构建产物共用一块盘，`docker/daemon.json:2` 的 `data-root=/root/docker` 也在同一分区。
- 备份走 `tar` 全量打包（`backup.mjs:102`），媒体增长后每次备份 IO 与时长线性上涨；而它以 `timeout: 120_000` 同步跑在请求线程里（`backups/route.ts:110-119`）——媒体一变大，管理员点「立即备份」就是一次 120 秒的请求超时。

**③ 建议**
给出：拆分后容器数/内存/构建时间/磁盘占用估算，磁盘水位阈值与告警；「立即备份」改为异步任务（复用 `background_tasks`），不在请求线程同步执行。

---

## S-10【P2】「独立迭代」的运维成本被忽略：两套版本、两套回滚、两套值班

**① 问题**
`proposal.md:18`「后台独立迭代」被列为收益，只字未提代价。

**② 后果**
本仓的发布单元是一个 compose 项目（`compose.yaml:23` 单一 name、单镜像 `dsh-plugin-review:local` 被 migrate/web/worker 共用 `compose.yaml:32`）。拆成两个应用后：镜像与版本 1→2、回滚点 1→2、依赖顺序从「migrate → web/worker」变成「migrate → web/worker 与 admin 的兼容窗口」。「独立迭代」的真实含义是**任意一侧单独升级都要考虑另一侧的 schema 兼容**，这与「后台功能越来越多」的初衷**部分反向**。

**③ 建议**
若动机只是「后台代码太多拖慢主站构建」，先做**同一镜像内的路由级拆分 / 构建产物分包**（零部署成本），把「双部署单元」留给确有独立资源或独立发布节奏需求时。方案里应附一份「拆 vs 不拆」的运维成本对照。

---

## 总评与放行条件

方案的问题不在细节，而在于**它把一个部署拓扑决策写成了三行配置改动**，并且没有提供任何运维面（备份、恢复、发布、回滚、容量、监控）。本仓已有的运维约束全是显式的、带注释的、被事故验证过的：唯一入口 nginx（`conf.d/default.conf:1-5`）、唯一迁移进程（`compose.yaml:1-8`）、备份单源（`docs/备份与恢复.md:20-32`）、`TRUST_PROXY` 与端口暴露的耦合（`compose.yaml:57-59`）。方案对这几条一条都没有回应。

**放行条件（缺一不可）**
1. 部署拓扑图 + 唯一入口/唯一迁移/唯一备份三个「唯一」的落点；
2. 库与备份路径写绝对路径，空库自检，恢复 runbook 覆盖全部单元；
3. 新单元的心跳/健康检查/重启策略 + busy 计数与 5xx 率两个观测点；
4. 域名/证书/OAuth 变更清单 + 5 分钟回切命令序列；
5. 资源预算与「主站 P95 / busy 率」两条可测判据。

在此之前建议**不进入实施**。
