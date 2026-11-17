# n29-graceful-shutdown「服务优雅停机改造」方案评审

> 评审对象：`atb/tasks/n29-graceful-shutdown/environment/proposal.md`（25 行）
> 核实方式：逐条对照 `server.mjs` / `src/worker/main.ts` / `docker/*` / `deploy/*` / `docs/项目规划/依赖与Linux部署.md`（只读取证，未改源码）

## 结论先行

方案要解决的问题**在本仓库已经解决**，且现有实现比方案更强：
- `server.mjs:98-115` 已有 `SIGTERM/SIGINT` → `closing` 幂等 → `server.close(cb)`（回调=最后一个连接关闭）→ 10s 兜底 `.unref()`，并打了三条结构化日志；
- `docker/web.Dockerfile:82` 已用 `tini` 做 PID1 转发信号；`docker/compose.yaml:69` web 已经是 `stop_grace_period: 15s`（方案 §3 是把"现状"当成"要改的"）；
- `src/worker/main.ts:248-255,272` 已有 SIGTERM → `running=false` → 主循环退出 → `heartbeat.stop()` + `closeDb()`。

按方案落地不是"新增一块"，而是**用更弱的机制替换更强的一套**。以下按严重度列问题清单。

---

## P0（阻塞：照方案执行会造成比现状更糟的故障）

### P0-1 §1「监听信号」重复实现且更弱，核心是 `process.exit(0)` 硬切断在途请求
**① 问题**：方案 `process.on('SIGTERM', async () => { server.close(); await sleep(5000); process.exit(0); })`（proposal.md:10-14）。
**② 后果**：
- `process.exit()` 会**立即终止**进程，不等事件循环排空 —— 到点那一刻仍在处理的请求被**连接重置**，与「重启不丢请求」（proposal.md:24）目标相反。本仓存在必然超过 5s 的请求：`src/server/platform/db.ts:54-56`（`synchronous=FULL` + `busy_timeout=5000`，单个等锁请求就能吃满 5s）、`proxy_read_timeout 60s`（`docker/nginx/snippets/proxy-common.conf:41`）、批量导入串行打 GitHub 是分钟级。
- 若只**新增**监听器而不删 `server.mjs:114-115`，会形成**两组互不感知的 SIGTERM 处理器**：新处理器 5s 到点先 `exit(0)`，旧实现的 `server_shutdown_done` 与收尾永不执行 → 进程以退出码 0 结束，日志上显示"优雅退出"，实际是中途硬杀 —— **故障被伪装成成功**，排障时最难定位的一类。
- 现有 `server.mjs:103/105/110` 的 `server_shutdown_start / done / timeout` 三条可观测信号随替换一并丢失，停机时序变成黑盒。
**③ 建议**：**不新增骨架**，直接复用 `server.mjs:100-113`：保存 `server.close(cb)` 的 callback 语义（最后连接关闭才 done），兜底用 `setTimeout(...).unref()`；如需改时长，只调 `server.mjs:112` 的 `10_000` 常量，且必须 `< stop_grace_period`。同时把「信号处理器唯一性」写成不变量（`process.once` 语义或替换而非叠加）。

### P0-2 方案未覆盖 worker 进程（`docker/compose.yaml:78-94`），停机语义最坏
**① 问题**：proposal.md 通篇只谈 web（"给 5 秒让进行中的请求完成"），而生产有两个写入进程。
**② 后果**：
- worker 收到 SIGTERM 后 `stop()` 只翻标志（`src/worker/main.ts:249-253`），只在 `runOnce` 之间生效（`src/worker/main.ts:202,211`）——**正在执行的 handler 不会被中断**；单任务上限 `TASK_DEADLINE_MS=300s`（`src/server/modules/tasks/constants.ts:45`）远大于 `stop_grace_period: 30s`（`docker/compose.yaml:94`），所以**每次部署都必然在 30s 时被 SIGKILL**。
- 被杀的 running 任务要等租约 `LEASE_DURATION_MS=120s`（`constants.ts:19`）过期才被 `recoverExpired`（`src/server/modules/tasks/service.ts:402-476`）回收，且**租约过期消耗重试预算**（service.ts:421-424），重启几次就可能把任务直接打成 `failed`。而 `main.ts:283-284` 的 `closeDb()` 在 SIGKILL 路径上**永远不会执行**。
- 权限面后果：发布/媒体任务在"文件已落地、库里未提交"的中间态被杀，会留下孤儿媒体文件与与 `/media` 权限判定不一致的条目（`docker/nginx/snippets/site-locations.conf:29-36` 的可见性由应用按库状态判定）。
**③ 建议**：方案必须把 worker 写成与 web 并列的第二个处理对象，给出：① 收到信号后停止 claim（已有）；② 对在跑任务设**软截止**——到 `< stop_grace_period` 时主动 `finish(outcome:'error', errorCode:'shutdown')`，让任务回到 `retry_wait` 而不是变成租约过期；③ 心跳 `detail` 里标 `stopping: true`，并把 `stop_grace_period(30s)` 与 `TASK_DEADLINE_MS(300s)` 的冲突写进方案。

### P0-3 §3 `stop_grace_period: 15s` 不是"要设的值"，是"已经设好的值"，且三个时限没有出处保证
**① 问题**：proposal.md:21 当作新增配置；而 `docker/compose.yaml:69` 已经是 `15s`。
**② 后果**：
- 方案的改动描述与仓库事实不符（前提核实失败），实施者会以为"改了一处"，实际什么都没改，问题继续存在。
- 三个时限互不相关且无推导：§2 的 5s、`server.mjs:112` 的 10s 兜底、`compose.yaml:69` 的 15s、`proxy_read_timeout 60s`（`proxy-common.conf:41`）。**60s > 15s** 意味着"存在合法的长请求一定会被停机切断"——这是方案没有回答的核心问题。
- 只用 compose 覆盖到一半部署面：`deploy/dsh-review-web.service` / `dsh-review-worker.service` **没有任何 `TimeoutStopSec`**，systemd 默认 90s，与 compose 的 15s/30s 差 3~6 倍，两套部署的停机行为不一致。
**③ 建议**：方案画出**停机时间预算表**并给出出处：`nginx 摘流窗口 → 最长可等请求 ≤ 应用内部兜底(10s) < stop_grace_period(15s) < 编排/反代层面的外层超时`；`deploy/*.service` 显式补 `KillSignal=SIGTERM` + `TimeoutStopSec=`（与 compose 对齐，并注明"两处必须同步改"）。

### P0-4 §3 只加 `stop_grace_period`，缺前置的**摘流**动作，nginx 会继续把新请求灌进正在退出的 web
**① 问题**：proposal.md:20-21 只改 compose 一个参数。
**② 后果**：`docker/nginx/conf.d/default.conf:14-22` 的 upstream 有 `zone` + `resolve` + `keepalive 16`，`nginx.conf:41` `resolver ... valid=10s` —— 后端换 IP 后 **最长 10s 内** nginx 仍会按旧地址转发；而 `server.close()` 只是"不再 accept"，既有 keepalive 连接上**新发起的请求**照常进入。`/healthz`（`site-locations.conf:8-12`）是 nginx 自己 `return 200`、**不回源**，所以编排器根本看不出 web 在退出。结果是：停机期间仍持续有请求打到"已 close 但尚未退出"的进程，反而**拉长**退出时间并触发 P0-1 的硬切断。
**③ 建议**：停机流程写成**三步**且顺序固定：① 置 `draining=true`（`server.mjs` 顶部一个模块级标志） → ② `/api/health` 在 draining 时返回 503（`src/server/platform/health.ts:150-152` 的 `statusForHealth` 加一条 `draining` 判据），让编排器/探针摘流，并给 nginx 留 ≥ `resolver valid=10s` 的观察窗口 → ③ 再 `server.close()` + 等回调。方案里要写明"先摘流、再等在途、最后退出"。

---

## P1（高：会造成明确的性能/正确性损失）

### P1-1 §2「固定 sleep 5s」是反模式：既拖慢每次部署，又给长请求设了隐性死线
**① 问题**：proposal.md:12,17-18。
**② 后果**：`server.close(cb)` 的回调在**最后一个连接关闭时**触发（`server.mjs:104-107` 已在用），比固定等待准确得多。固定 5s 的两种失败方向都成立：常态下**白白多耗 5s**（即使第 0.1s 就没有在途请求）；异常下 5s 不够就**硬切**。全站每次 `docker compose up -d` / systemd restart 都要付这 5s。
**③ 建议**：以"活跃连接归零"为准，固定时间只作为**兜底上限**存在，并且必须是可观测的（`server_shutdown_timeout` 该长期为 0；出现的次数应作为告警）。

### P1-2 `socket.setKeepAlive`/空闲 keepalive 连接会把退出拖到兜底时限
**① 问题**：方案没有 `closeIdleConnections` 的概念。
**② 后果**：`proxy-common.conf:37-38` 用 `Connection ""` + HTTP/1.1 复用了 nginx→web 的长连接，`keepalive_timeout 65`（`nginx.conf:34`）。空闲 keepalive 连接**不会自己关**，`server.close()` 等的是"所有连接"，于是每次停机都会**卡满兜底时限**才退出（现有 `server.mjs` 的 10s 兜底恰好掩盖了这一点，表现为"每次重启都要 10 秒"）。
**③ 建议**：在 `shutdown()` 里显式 `server.closeIdleConnections()`（Node 18+），只对空闲连接生效；在途请求仍由 `close()` 回调等待。

### P1-3 web 侧从不 `closeDb()`，`synchronous=FULL` 下的收尾没有被纳入停机流程
**① 问题**：方案只碰 `server.mjs` 与 compose。
**② 后果**：全仓只有 `src/worker/main.ts:284` 与 `src/server/platform/migrate-cli.ts` 调用 `closeDb()`；web 的 `getDb()` 单例（`src/server/platform/db.ts:67-74`）进程退出时直接由内核释放。WAL + `synchronous=FULL`（`db.ts:53-55`）下一般不会损坏，但"最后一个事务是否落盘"没有被任何代码保证，`heatbeat`/备份等收尾也没有顺序可言。
**③ 建议**：在 `server.close` 回调里、`process.exitCode = 0` **之前**加 `closeDb()`；兜底超时路径先记 `server_shutdown_timeout` 再 `exitCode = 非0`。

### P1-4 兜底路径用 `process.exit(0)`/退出码 0，把"没退干净"伪装成成功
**① 问题**：proposal.md:13；现状 `server.mjs:109-112` 同样只设 `exitCode = 0`。
**② 后果**：超时被 SIGKILL 与"优雅退出完成"在编排器/监控侧**无法区分**；`Restart=always`（`deploy/dsh-review-web.service:20-21`）与 `restart: unless-stopped`（`compose.yaml:68`）也不做区分。"以 0 退出"是对"成功"的错误声明。
**③ 建议**：超时兜底走 `process.exit(1)`（或专用退出码），并保留 `server_shutdown_timeout` 日志作为唯一可判据信号；正常路径沿用 `exitCode = 0` + 自然退出。

### P1-5 worker 在 systemd 下可能**收不到** SIGTERM（tsx 子进程链）
**① 问题**：方案与两个 unit 文件都没处理。
**② 后果**：`deploy/dsh-review-worker.service:17` 是 `node .../tsx/dist/cli.mjs src/worker/main.ts`——tsx CLI 拉起的**子进程**才是真正跑 worker 的进程；systemd `Type=simple` 默认只把 SIGTERM 发给主进程（cgroup 级 kill 只在超时后）。若 tsx 未转发，`src/worker/main.ts:254` 的监听器**根本不会触发**，`TimeoutStopSec` 到点整组被杀 → 与 P0-2 的最坏路径叠加。
**③ 建议**：要么 worker 改用编译产物直接 `node dist/worker/main.js`（去掉中间进程），要么 unit 用 `KillMode=control-group` + 显式 `TimeoutStopSec`；方案需写明"信号必须直达业务进程"这一前提，并给出验证方法（`journalctl` 要能看到 `收到退出信号，准备优雅退出`）。

### P1-6 部署顺序未纳入：`migrate` 是 oneshot，重启窗口里新旧进程会同时服务
**① 问题**：方案只改停机参数，没写"谁先停、谁后起"。
**② 后果**：`docker/compose.yaml:27-48` 中 `migrate` 是 `restart: 'no'` 的一次性容器，`web`/`worker` `depends_on: service_completed_successfully`。`docker compose up -d` 时 migrate 先跑、而**旧 web/worker 仍在服务新 schema**（0017 之类含 `PRAGMA foreign_keys=OFF` 的重建类迁移期间尤甚）。方案声称"部署更平滑"（proposal.md:25），这一条恰恰是部署不平滑的真实来源。
**③ 建议**：明确顺序 `摘流 → 停 worker → 停 web → migrate → 起 worker/web → 验证 → 开流`，与 `docs/项目规划/依赖与Linux部署.md:100-104`（"入口进入维护状态，停止接收新写入及领取任务"）保持一致，并把这条写进运维手册。

---

## P2（中：可观测性 / 边界语义缺失）

### P2-1 停机期间错误语义未定义
**① 问题**：方案没有规定退出过程中到达的请求返回什么。
**② 后果**：`docker/nginx/snippets/proxy-common.conf:39-41` 对 upstream 断连**没有配 `proxy_next_upstream` 重试**，客户端直接看到 502/连接重置；而 `status >= 500` 在 `server.mjs:67-68` 会被记成 `warn`，与真正的服务端错误混在一起，监控误报。
**③ 建议**：draining 期间对**新到**请求直接返回 `503 + Retry-After`（或 `Connection: close`），并在访问日志里给独立的标记字段（如 `draining:true`），使"停机导致的 5xx"与"代码 bug 的 5xx"可区分。

### P2-2 `process.exit(0)` 会吞掉未 flush 的日志
**① 问题**：proposal.md:13；`process.exit()` 不等 stdout/stderr 排空。
**② 后果**：管道/容器 json-file 驱动（`compose.yaml:70-74`）下，最后几行**正是停机日志**，丢了就没有任何证据说明这次退出了没有。`src/server/platform/log.ts` 与 `server.mjs:41-42` 都是裸 `process.stdout.write`。
**③ 建议**：兜底路径不要 `process.exit`，只设 `exitCode` 并 `server.unref()` + `clearInterval` 让事件循环自然枯竭；真要强退，先确认最后一条 `server_shutdown_timeout` 已写出。

### P2-3 验收判据不可测量
**① 问题**：proposal.md:23-25 只有两句"预期"。
**② 后果**：`atb/tasks/n29-graceful-shutdown/tests/criteria.md` 要求判据可核对，而方案没有给任何可验证信号；部署后"有没有变好"无法回答。
**③ 建议**：以既有日志作为验收标准——每次部署 `journalctl -u dsh-review-web | grep server_shutdown` 必须看到 `start → done`，且 `done - start < 8s`、`server_shutdown_timeout` 出现次数长期为 0；worker 侧验收 `收到退出信号，准备优雅退出` 出现且进程在 grace 内自然退出（无 SIGKILL）。

### P2-4 未定义 SIGINT / 二次信号 / 幂等
**① 问题**：方案只有单次 SIGTERM，无 `closing` 标志。
**② 后果**：重复信号会重入（第二次 Ctrl-C 又跑一遍 `close + sleep + exit`），且 SIGINT 不在覆盖范围。现状 `server.mjs:99-102` 已有 `closing` 守卫、`:114-115` 同时监听两个信号——方案是把已有的幂等性丢掉了。
**③ 建议**：保留 `closing` 幂等；二次信号可直接升级为"立即强退 + 记 warn"，这是安全的做法。

---

## 建议的落地口径（替代方案）

1. **不新增信号处理器**：以 `server.mjs:98-115` 为唯一实现，逐项补强而不是重写。
2. `shutdown(signal)` 内部顺序：`closing=true` → 记 `server_shutdown_start` → 置 `draining`（health 转 503，留出 ≥ nginx `resolver valid=10s` 的摘流窗口） → `server.closeIdleConnections()` → `server.close(cb)` → cb 内 `closeDb()` + `server_shutdown_done`；兜底 `< stop_grace_period` 定时器 `.unref()`，触发时记 `server_shutdown_timeout` 并 `exitCode = 1`。
3. worker：停止 claim + 在跑任务软截止 `finish(errorCode:'shutdown')` + 心跳标 `stopping`；解决 tsx 信号转发。
4. 部署面：`compose.yaml:69,94` 与 `deploy/*.service` 的 `TimeoutStopSec` 用**同一张预算表**推导并双向同步；写明 migrate 与停启顺序。
5. 验收：三条 shutdown 日志 + worker 退出日志作为每次部署的必查项。

> 若结论是"现状已够、只差参数与 worker 软截止"，建议把本任务改判为**现状核实 + 小步补强**，而不是按原方案重写停机逻辑。
