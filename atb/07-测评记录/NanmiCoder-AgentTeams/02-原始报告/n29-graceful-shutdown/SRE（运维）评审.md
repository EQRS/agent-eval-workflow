# n29-graceful-shutdown SRE（运维）评审

> t30 · e3-fixer · 2026-11-17

---

n29-graceful-shutdown SRE 视角评审完成。落盘：atb/tasks/n29-graceful-shutdown/review-sre.md（133 行，只读评审，未改源码）。

结论：方案描述的东西在仓库里已经实现了，而且实现得比方案更好——server.mjs:98-115 已用 server.close() 回调 + 10s 兜底（.unref），web.Dockerfile:82 有 tini 转发 SIGTERM，compose.yaml:69 已经是 stop_grace_period: 15s。照抄落地等于用 sleep 替换现有回调，属倒退。真正缺的三件事方案一件没写：nginx 摘流、健康检查 draining、worker 停机时序。

摘要前 5 条：
1.【严重】重复且更差：方案把「等最后连接关闭」换成「固定 sleep 5000 再 process.exit(0)」。db.ts:56 busy_timeout=5000ms，单个等锁请求就能吃掉 5 秒；批量导入是分钟级。且现有 server_shutdown_start/done/timeout 三条结构化日志会被覆盖，停机时序反而不可观测。
2.【严重】无摘流：nginx upstream dsh_web 有 keepalive 16（docker/nginx/conf.d/default.conf:14-22）+ Connection ""（proxy-common.conf:37-38）。server.close() 只停 accept 不切已建连接，那 5 秒正是「旧连接继续投递」的窗口 → 随机 502。建议 健康检查转 503 → 等 keepalive 周期 → 再 close。
3.【严重】停机期间 /api/health 仍 200：web.Dockerfile:76-77 容器 HEALTHCHECK 打它，health.ts:56-90 只看库/磁盘、没有 draining 维度 → 编排器与探针都被骗过。建议 draining 进 health 报告。
4.【高】未提 worker 容器（compose.yaml:78-94，stop_grace_period: 30s）：main.ts:248-255 的 SIGTERM 只在 runOnce 之间生效，在跑任务不被中断，30s 后 SIGKILL → 任务留 running 状态等 120s 租约过期才被 recoverExpired 回收（service.ts:402-440），期间占并发位；closeDb()（:284）也不会执行。
5.【中】时间预算无出处：5s（方案）/10s（server.mjs 兜底）/15s（compose）、nginx proxy_read_timeout 60s（proxy-common.conf:41）四者互不相关；60s > 15s 意味着必然存在被切断的长请求，方案没有一条推导。

完整清单 S-01…S-08，另含：部署顺序缺一节——compose.yaml:33/46-48 的 migrate 是 oneshot 且 web depends_on 它，重建类迁移（0007 那种 PRAGMA foreign_keys=OFF）期间旧 web 仍在服务新 schema；无停机验收信号（应把三条 shutdown 日志列为必查项、server_shutdown_timeout 长期为 0）；process.exit(0) 会吞掉异步 stdout 里最后几行停机日志（现有实现用 process.exitCode 是对的）。

最小改动 5 条：保留 server.mjs 机制只加 draining 标志；health 在 draining 时返回 fail；worker 停止 claim + 在跑任务 30s 软截止 finish；写明 migrate→web→worker 部署顺序；三条 shutdown 日志作为验收。

前提核实：§3 的 stop_grace_period: 15s 与仓库现状一致（compose.yaml:69），§1 的 SIGTERM/server.close 在 server.mjs:100-115 真实存在——方案的前提不是「不存在」，而是「已存在且更完整」。