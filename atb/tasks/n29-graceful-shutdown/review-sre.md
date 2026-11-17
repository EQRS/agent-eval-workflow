# 评审：服务优雅停机改造 · SRE（运维）视角

> 评审对象：`atb/tasks/n29-graceful-shutdown/environment/proposal.md`（25 行）
> 评审人身份：**SRE / 运维**（只看停机时序、信号链、连接排空、迁移顺序、可观测与回滚；不评业务功能）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**这个方案描述的东西在仓库里已经实现了，而且实现得比方案更好**——`server.mjs:98-115` 已经在用 
> `server.close()` 回调 + 10 秒兜底，`docker/compose.yaml:69` 也已经是 15s。方案如果照抄落地，是**用 sleep 替换掉现有的关闭回调**，属倒退。
> 真正没做的是另外两件事：**nginx 侧没有摘流、worker 侧没有停机时序**——方案一个字没提。

---

## 摘要（前 5 条最重要）

1. 【严重】**方案要解决的「停机丢请求」在本仓库已经解决，且现有实现优于方案。**
   依据：`server.mjs:98-115`（`closing` 幂等标志 → `server.close(cb)`，回调里记 `server_shutdown_done` → `process.exitCode = 0`；
   `:109-112` 10 秒兜底定时器 + `.unref()`）；`docker/web.Dockerfile:82`（`ENTRYPOINT ["/usr/bin/tini", "--"]`，注释写明「tini 只负责转发信号 + 回收僵尸，SIGTERM 才能真的传到 node」）；
   `docker/compose.yaml:69`（web 已经是 `stop_grace_period: 15s`）。
   后果（如果照方案落地）：把「等最后一个连接自然关闭」换成「固定 sleep 5000 再 `process.exit(0)`」——
   **凡是超过 5 秒的请求都会被硬切**（本仓 `busy_timeout = 5000ms`，`src/server/platform/db.ts:56`，一个等锁的请求单独就能吃掉 5 秒；
   批量导入这类串行打 GitHub 的请求更是分钟级）。而且现有的 `server_shutdown_done / server_shutdown_timeout` 两条日志会被新实现覆盖掉，**停机时序反而变得不可观测**。
   建议：**删掉 §1**，以 `server.mjs:98-115` 为唯一实现；若要改，只改超时时长，不要改机制。

2. 【严重】**没有任何「先摘流再停机」的步骤——nginx 侧 16 条 keepalive 长连接会继续把请求打到正在关闭的 web。**
   依据：`docker/nginx/conf.d/default.conf:14-22`（`upstream dsh_web { zone …; server web:3000 resolve; keepalive 16; }`）、
   `docker/nginx/snippets/proxy-common.conf:37-38`（`Connection ""` + HTTP/1.1 → 复用长连接）。
   后果：`server.close()` 只停止 accept，**不切断已建立的连接**；nginx 复用的 keepalive 连接会被继续用来转发新请求，
   web 在那 5 秒（或 15 秒）里既在「关闭中」又在「接新请求」，直到进程退出把连接掐断——用户看到的是随机的 502/连接重置。
   方案里 `server.close()` 之后 `await sleep(5000)` 的这 5 秒，恰恰是「旧连接仍在送新请求」的窗口。
   建议：停机顺序必须是 **① 健康检查立即转 503（见摘要 3）→ ② 等待一个大于 nginx `keepalive` 周期的排空窗口（如 5-10s）→ ③ 再 `server.close()`**；
   或者在 web 侧主动给 `Connection: close` 响应头（逐步退掉 keepalive），让 nginx 自然换新连接。

3. 【严重】停机期间 `/api/health` 仍然返回 200，编排器与探针都被骗过。
   依据：`docker/web.Dockerfile:76-77`（容器 HEALTHCHECK 打 `/api/health`，失败 3 次才判 unhealthy，间隔 30s）；
   `src/server/platform/health.ts:56-90`（`buildHealthReport` 只看库/磁盘，**没有「正在关闭」这一维**）。
   后果：`docker compose stop` / 滚动重启时，Docker 在 stop_grace_period 内不会因为健康检查而提前判定「不可用」，
   外部探针（如果有 LB 或监控）同样看到 200；这与 C-4 是同一件事：**没有 drain 状态，就没有摘流信号**。
   建议：`shutdown` 一开始就置一个进程级 `draining=true`，`buildHealthReport` 在这里返回 `status='fail'`（或新增 `draining` 检查项）；
   同时把容器 HEALTHCHECK 的 `--interval` 视为已知延迟来源，明确「摘流窗口 ≥ 一个 interval」的取值依据。

4. 【高】完全没提 worker 容器：它才是「停了会留下脏状态」的那个进程。
   依据：`docker/compose.yaml:78-94`（worker 是独立服务，`stop_grace_period: 30s`）；
   `src/worker/main.ts:248-255`（SIGTERM → `running = false`）、`:280-285`（`await worker.run()` 退出后 `heartbeat.stop()` + `closeDb()`）；
   `src/server/modules/tasks/constants.ts:19,45`（租约 120s、单任务截止 300s）；`src/server/modules/tasks/service.ts:402-440`（`recoverExpired`）。
   后果：worker 收到 SIGTERM 后**只会在「下一轮循环」退出**（`shouldContinue` 在 `runOnce` 之间检查），
   而一个正在执行业务的任务（发现/发布/批量）不会被中断——`stop_grace_period: 30s` 到期即 SIGKILL，
   该任务停在 `running` 状态、租约未释放，要等 `recoverExpired` 按 120s 租约 + 重试预算回收（期间它一直占着并发位，见 `constants.ts:16`）。
   更糟的是 SIGKILL 时 `closeDb()`（`:284`）不会执行 → SQLite 连接被粗暴关闭（WAL 模式下通常安全，但 `-wal`/`-shm` 残留要等下次 checkpoint）。
   建议：worker 侧补三件事——① 收到信号后**不再 claim 新任务**（已有）；② 给正在跑的任务一个「软截止」：到 30s 时主动 `finish(outcome:'error', errorCode:'shutdown')` 让任务回到 `retry_wait` 而不是租约过期；
   ③ 把 `stop_grace_period` 与 `TASK_DEADLINE_MS` 的关系写进方案（30s 远小于 300s，本来就是不匹配的）。

5. 【中】§3 的 `stop_grace_period: 15s` 与 §2 的 5 秒 + `server.mjs` 的 10 秒兜底三者数字互不相关，没有一个出处。
   依据：`proposal.md:18,21`；`docker/compose.yaml:69`（web 15s）、`:94`（worker 30s）；`server.mjs:109-112`（10s 兜底）。
   后果：5 < 10 < 15 只是「碰巧不冲突」：关闭回调要等连接关完，兜底 10s 先到就先退出，
   15s 是容器层的最后保险。读者无法判断「哪个数字是该调的」。
   建议：在方案里画一条**停机时间预算表**：摘流窗口 + 请求最长耗时（可由 `proxy_read_timeout 60s` 推导，`proxy-common.conf:41`）≤ `stop_grace_period`；
   并明确「超过预算的请求允许被切断，但必须先写日志留证」。

---

## 完整问题清单（按严重度）

### S-01 【严重】方案与既有实现重复且更差（sleep vs close 回调）
- 现场：`proposal.md:9-18`。
- 依据：`server.mjs:98-115`；`docker/web.Dockerfile:82-83`；`docker/compose.yaml:69`。
- 后果：见摘要 1。补充：`server.close()` 的回调只在**所有连接关闭后**触发，方案却用它之后又 sleep——
  即在「已经安全」之后再多等 5 秒，然后在「还不一定安全」的时刻硬退出（sleep 与连接状态无关）。
- 建议：保留现有实现；若一定要改，只调 `server.mjs:112` 的 10_000 常量，并保持 `< stop_grace_period`。

### S-02 【严重】缺少摘流（drain）阶段：nginx keepalive 会继续投递
- 现场：`proposal.md` 全文无 nginx / 摘流相关内容。
- 依据：`docker/nginx/conf.d/default.conf:14-22`（`keepalive 16`）、`proxy-common.conf:37-38`（HTTP/1.1 + `Connection ""`）。
- 后果：见摘要 2。
- 建议：健康检查转 503 → 等 `keepalive` 周期 → `server.close()`；或对即将关闭的进程返回 `Connection: close`。

### S-03 【严重】健康检查没有 drain 状态
- 现场：`proposal.md:10-14`（直接 sleep，期间健康检查照常 200）。
- 依据：`docker/web.Dockerfile:76-77`、`src/server/platform/health.ts:56-90`。
- 后果：见摘要 3。
- 建议：进程级 `draining` 标志进 health 报告；容器 HEALTHCHECK 的 30s 间隔要吃进排空窗口的预算里。

### S-04 【高】worker 容器缺失停机时序
- 现场：`proposal.md` 只谈 web（「服务」一词在容器化部署里指哪个进程没写清）。
- 依据：`docker/compose.yaml:78-94`；`src/worker/main.ts:248-255,280-285`；`src/server/modules/tasks/constants.ts:16,19,45`；`service.ts:402-440`。
- 后果：见摘要 4。
- 建议：worker 侧「停止 claim + 软截止 finish + 让租约被干净释放」；`stop_grace_period: 30s` 与 300s 任务截止的关系必须写明。

### S-05 【中】时间预算无出处，三个数字互不相干
- 现场：`proposal.md:18,21`。
- 依据：`compose.yaml:69,94`；`server.mjs:109-112`；`docker/nginx/snippets/proxy-common.conf:39-41`（connect 5s / read 60s）。
- 后果：见摘要 5。
- 建议：给出「摘流窗口 + 最长请求耗时 ≤ stop_grace_period」的推导；把 60s 的 `proxy_read_timeout` 纳进来（它比 15s 的宽限期还长，意味着**必然存在被切断的长请求**）。

### S-06 【高】部署链路里还有第三个容器：`migrate` 是 oneshot，重启顺序会决定「旧 web + 新 schema」的窗口
- 现场：`proposal.md` 无部署顺序说明。
- 依据：`docker/compose.yaml:33`（`migrate` 服务 `command: npx tsx src/server/platform/migrate-cli.ts`）、`:46-48`（web `depends_on: migrate: service_completed_successfully`）；
  `migrations/0007_task_type_plugin_discovery.sql:1-19`（重建表类迁移会 `PRAGMA foreign_keys=OFF`，中途失败留半成品，`src/server/platform/migrate.ts:46-47`）。
- 后果：`docker compose up -d` 时 migrate 先跑完，**旧 web 仍在服务新 schema** 直到被替换；若这次迁移是重建类（0007 那种），
  旧代码在新表结构上运行的那几十秒是整个部署里最脆弱的窗口，而优雅停机方案对此无话可说。
- 建议：在方案里补「部署顺序」一节：先 `migrate` → 再滚动替换 web（旧实例先摘流再退出）→ 最后替换 worker；
  重建类迁移期间明确标注「web 应停或只读」。

### S-07 【中】没有「停机是否成功」的验收信号
- 现场：`proposal.md:23-25`（预期只有两句）。
- 依据：`server.mjs:103-111` 已打 `server_shutdown_start / server_shutdown_done / server_shutdown_timeout` 三条结构化日志；
  `deploy/dsh-review-web.service:23-25`（`StandardOutput=journal`）。
- 后果：改造后如果丢掉了这三条日志（换成 sleep 实现），运维就再也无法回答「这次重启有没有请求被截断」。
- 建议：把这三条日志列为**验收标准**：每次部署后 `journalctl -u dsh-review-web | grep server_shutdown` 必须看到 start→done，
  `server_shutdown_timeout` 出现次数应长期为 0；同时（可选）在 web `res.on('close')` 里统计未完成请求数并打一条汇总。

### S-08 【低】`process.exit(0)` 会吞掉未 flush 的日志
- 现场：`proposal.md:13`。
- 依据：`server.mjs:39-43`（日志用 `process.stdout.write` 直接写，命中管道时是异步的）、`:106`（现有实现用的是 `process.exitCode = 0`，**让事件循环自然退出**）。
- 后果：`process.exit(0)` 立即终止，管道中尚未写出的最后几行（正是停机日志）可能丢失——恰恰是最需要留证的时刻。
- 建议：与现有实现对齐，只设 `process.exitCode`；必须强制退出时先 `await` 一次 stdout flush。

---

## 结论与建议

**方案不应对照落地**。它描述的机制在本仓已经存在（`server.mjs:98-115` + tini + compose 15s），且现有实现比它正确；
真正缺的三件事方案一件没写：**摘流（nginx keepalive 16 条长连接）**、**健康检查的 draining 状态**、**worker 的停机时序**。

**建议的最小改动清单**：
1. web：`server.mjs` 的 `shutdown()` 开头置 `draining = true`（供 health 读取），`server.close()` + 回调 + 10s 兜底**全部保留**；不引入 sleep。
2. 摘流：`buildHealthReport` 在 `draining` 时返回 `fail`（`src/server/platform/health.ts:56-90` 加一项检查）；并确认排空窗口 > nginx `keepalive` 复用周期。
3. worker：SIGTERM 后停止 claim（已有）+ 对在跑任务软截止（30s 时 `finish(errorCode:'shutdown')`），避免租约过期留 `running` 脏行。
4. 部署：写明 migrate → web → worker 的顺序与「重建类迁移期间 web 停或只读」。
5. 验收：`server_shutdown_start/done/timeout` 三条日志作为每次部署的必查项；`server_shutdown_timeout` 应长期为 0。

> 评审人声明：本报告只覆盖运维/停机时序视角；业务语义与接口契约由其他视角负责。

> 前提核实：方案 §3 的 `stop_grace_period: 15s` **与仓库现状一致**（`docker/compose.yaml:69`，即这一条是「已经是这样」而非「要改成这样」）；
> §1 的 `process.on('SIGTERM')` 与 `server.close()` 在 `server.mjs:100-115` 真实存在。因此方案的前提不是「不存在」，而是「已存在且更完整」。
