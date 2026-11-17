# n29-graceful-shutdown · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0007
> 模型 deepseek-chat · 9200 字 · 29 次工具调用

---

所需材料已经齐全。开始撰写评审。

---

# SRE 评审：n29 服务优雅停机改造

## 先说结论

**方案与现有代码严重脱节，如果按字面实施，会退化现有实现。** 仓库里 `server.mjs:95-113` **已经实现了** SIGTERM/SIGINT 优雅退出（`server.close()` + 10s 兜底 + `process.exitCode`），`src/worker/main.ts:225-236` 也已实现信号停止领取新任务。方案中的 `sleep(5000)` + `process.exit(0)` 若覆盖到 `server.mjs`，会让“优雅停机”变成“最多 5 秒服务、随后无条件强制退出”。

方案全文 25 行（`proposal.md:1-25`），**没有一行提到 worker、数据库收尾、nginx 摘流、systemd 那套部署**。因此以下按“缺少什么”来评审。

---

## 一、`sleep(5000)` + `process.exit(0)` 是最严重的一处

**问题部分**：`proposal.md:8-13`

```ts
process.on('SIGTERM', async () => {
  server.close();
  await sleep(5000);
  process.exit(0);
});
```

**后果**

1. **`server.close()` 不返回 Promise，也没有 `await`**（`proposal.md:10`）。它只是“停止接受新连接”，回调抵达时才知道在途请求是否全部完成。紧接着固定 `sleep(5000)`，语义从“等在途请求处理完”变成了“**最多等 5 秒，到点强制退出**”。
2. **`process.exit(0)` 会立即终止事件循环**（`proposal.md:12`）。第 5 秒执行到这一刻仍在处理的请求（导出、大文件上传、慢查询）会被直接终止——客户端收到连接重置。而“重启不丢请求”正是方案的预期（`proposal.md:23-24`），所以**方案自相矛盾**：它比现有实现更容易丢失请求。
3. **用 `process.exit(0)` 作为退出方式会掩盖失败**：即使所有请求都被强制终止，退出码仍是 0，编排器会认为这是一次干净停机，日志里看不到任何异常。
4. **覆盖风险**：`server.mjs:108` 的现有兜底超时是 10s，且配了 `.unref()`——它**不阻止进程自然退出**。而 `compose.yaml:75` 给 web 的 `stop_grace_period` 是 **15s**。三者当前是自洽的：进程最多 10s 就退出，编排器给 15s。方案改成 5s + `exit(0)` 后，这个自洽被破坏，且 15s 的编排器预算再也用不上。

**建议**

- **不要改 `server.mjs`**。它已经是对的：`server.close(callback)` 里收尾 + `process.exitCode = 0` + `setTimeout(...).unref()` 兜底（`server.mjs:95-113`）。方案应改为“**确认并文档化现有实现**”，而不是新增一套。
- 若确实要提炼，唯一需要补的是 `server.close()` 之后**主动清理空闲 keep-alive 连接**：`server.close()` 不关闭空闲长连接（`nginx.conf:41` 的 upstream `keepalive 16` + `proxy-common.conf:40` 的 `Connection ""` 保证连接会被复用），Node 会一直等到 `keepAliveTimeout` 才释放，可能把兜底超时耗尽。可参考 `server.closeIdleConnections()`（Node ≥18）在 `close()` 后调用，并给在途连接一个独立的 `closeAllConnections()` 最终截止时间。
- **删除 `process.exit(0)`**，一律用 `process.exitCode`。这是仓库里既有的纪律，见 `scripts/worker-healthcheck.mjs:12-13` 的同类注释（`process.exit()` 在 better-sqlite3 下会触发 libuv 断言）。

---

## 二、方案漏掉了 worker，而它是真正会丢数据的那一个

**问题部分**：`proposal.md:5`「1. 监听信号」只写了 `server`，全文未提 worker。

**后果**

- 本项目有两个常驻进程：`web`（`compose.yaml:44`）和 `worker`（`compose.yaml:79`）。**worker 才是持有“在途业务工作”的进程**——它 claim 任务、跑处理器、改业务表。
- worker 的优雅退出已有实现（`src/worker/main.ts:225-231` 设 `running=false`，`main.ts:210` 的 `shouldContinue` 让循环在**当前任务结束后**才退出）。这套是**正确但耗时不可控**的：`constants.ts:50` 规定单任务上限 `TASK_DEADLINE_MS = 300s`（5 分钟），而 `compose.yaml:98` 的 `stop_grace_period` 只有 **30s**。
- 也就是说：**worker 收到 SIGTERM 后，只要手上有个跑超过 30 秒的任务，就一定会被 SIGKILL 打断**。任务被硬杀后，`background_tasks` 里的租约要等 `LEASE_DURATION_MS = 120s`（`constants.ts:22`）到期，才由 `recoverExpired` 回收重跑。**这正是方案声称要解决的“重启中断在处理中的请求”，但方案完全没覆盖它。**
- 更糟的是重复执行：任务在 SIGKILL 时可能已经写了一半业务表（`handlers.ts` 里的发布流程），重跑时是否幂等取决于各 handler 自己。方案没有要求任何幂等性验证。

**建议**

- 方案必须**明确写出 worker 的停机语义**，并与 `stop_grace_period: 30s`（`compose.yaml:98`）对齐。两个选项，二选一并在方案里写明：
  - (a) **调大 budget**：把 worker 的 `stop_grace_period` 提到 > `TASK_DEADLINE_MS`（300s），代价是 `docker compose down` / 部署会等最长 5 分钟。
  - (b) **保留 30s，但保证任务可中断**：明确“worker 收到 SIGTERM 后 `shouldContinue` 立即为 false，在跑的任务若超过 grace 被 SIGKILL 是可接受的，因为 `recoverExpired` 会重跑”——**前提是每个 handler 幂等**。这需要在方案里列出幂等性依据（`operation_records` 的 `UNIQUE(action_type, operation_key)` 去重，见 `migrations/0042_audit_immutable.sql:5-7`），不能默认成立。
- 顺便记一条：worker 的 `sleep(IDLE_SLEEP_MS)`（`main.ts:216`，10s）在收到信号时**不会被唤醒**，最坏要空等 10 秒才检查 `shouldContinue`。30s 预算里先白白花掉最多 10s。建议 sleep 用可中断的定时器（收到信号即 resolve）。

---

## 三、SQLite 收尾没有任何约定（且现有代码有个真实缺口）

**问题部分**：方案对“停机时数据库怎么办”只字未提。

**后果**

- 全部状态在一个 SQLite 文件里（`compose.yaml:52` `DSH_REVIEW_DB_PATH=/app/.data/app.sqlite`），WAL 模式（`deploy/dsh-review-worker.service:6-7` 明确说“SQLite 在 WAL 下支持多进程”）。
- 强杀进程后 `-wal` / `-shm` 文件会遗留。WAL 本身能自恢复，但**备份链路会受影响**：`BACKUP_ROOT` + `BACKUP_KEEP=14`（`compose.yaml:61-62`）的每日备份若正好在强杀时段取快照，需要确认它是否用 SQLite backup API 而非直接 `cp`（直接 `cp` 一个活跃 WAL 库会拿到不一致副本）。
- **现有缺口**：`src/worker/main.ts:296-299` 的 `finally { heartbeat.stop(); closeDb(); }` 只在 `worker.run()` 正常返回时执行。**如果 worker 被 SIGKILL（grace 超时的必然结果），`closeDb()` 不会跑。** 这在 SIGKILL 下无解，但方案应明确记录：这是“预算倒挂”（第二节）的已知后果，而不是可忽略的细节。
- `server.mjs` 侧**没有**任何 `closeDb()`——web 进程退出时 DB 连接靠进程终止回收。这在 `process.exitCode` 优雅路径下也是把收尾交给 OS，`-wal` 是否 checkpoint 不受控。至少应在方案里声明“web 的 DB 收尾由进程退出兜底，不保证 WAL checkpoint”，别让运维以为是干净的。

**建议**

- 方案增加一节「**停机时的数据收尾**」：明确 web / worker 各自是否 `closeDb()`（worker 已有、web 没有），以及 WAL checkpoint 责任。
- 把“部署后必须验证”写成运维检查项：`docker compose restart web && ls -la .data/` 确认 `-wal`/`-shm` 正常回收。
- 核对备份脚本（`deploy/dsh-review-backup.service` + `scripts/` 下的 backup 实现）是否使用 backup API；若是 `cp`，方案里必须标注“重启期间不要触发备份”，或让备份加互斥。

---

## 四、`stop_grace_period: 15s` 已经存在，方案等于没提新增项

**问题部分**：`proposal.md:17`「compose 里设 `stop_grace_period: 15s`」

**后果**

- 这一条**已经是现状**：`compose.yaml:75`（web）就是 `stop_grace_period: 15s`。方案把既有配置当成“新增改造”，会让评审和验收都误判工作范围。worker 那边是 30s（`compose.yaml:98`），方案的 15s 说法**对 worker 不成立**。
- 更关键：**方案漏掉了编排器的停机顺序问题**。`compose.yaml:131-133` 里 nginx `depends_on: web: condition: service_healthy`——这只管**启动**顺序。停机时 Docker 是**并行**给各容器发 SIGTERM 的。后果是：web 已经在 `server.close()` 拒绝新连接，但 nginx 还在往它上面转流量（`default.conf:25-34` 的 upstream 仍解析到它），窗口期内出现 **502**。方案预期“部署更平滑”（`proposal.md:25`）在这里落空。
- 另有一条隐含风险：`nginx.conf:40` 的 `resolver 127.0.0.11 valid=10s` + `default.conf:28` 的 `resolve`。web 重建换 IP 后，nginx 最长有 **10 秒**仍指向旧 IP。这条注释（`nginx.conf:36-40`）记录了 2026-10-18 因此**全站 502** 的实战事故。方案完全没有触及。

**建议**

- 方案里把这一条改成**如实的差异清单**：web 15s（已存在）、worker 30s（已存在），并说明本次是否要调整及理由，而不是当成新配置。
- 补「**nginx 摘流先于 web 停机**」：给出可执行做法，例如部署脚本里先 `docker compose stop nginx`（或 `nginx -s quit` + 让 `docker compose ps` 确认），再停 web；或者给 web 加 `pre_stop` 钩子/健康检查摘除。若不做，方案必须承认“重启窗口内仍可能有 5xx”，把预期从“不丢请求”下调为“**大幅减少丢请求**”。
- 与 `resolver valid=10s` 的 10 秒重解析窗口对齐：明确部署后 nginx 是否需要 `nginx -s reload`，以及验收时如何确认不再 502。

---

## 五、静默：健康检查会把正常停机误报成故障

**问题部分**：方案未提健康检查与停机的交互。`compose.yaml:44-77` 的 web 没有 `healthcheck` 段（健康检查写在镜像里，`web.Dockerfile:73-74`，`HEALTHCHECK` 打 `/api/health`）；worker 的健康检查是 `scripts/worker-healthcheck.mjs`。

**后果**

- web 进入 `server.close()` 后，`/api/health` 会**立刻拒绝连接** → HEALTHCHECK 失败。若停机被拉长（本次兜底 10s，`server.mjs:105`），连上 `interval 30s` / `retries 3` 的边缘就会在停机过程中被标记 unhealthy。对外部编排器（将来上 K8s/负载均衡）来说，这会触发“不健康即重启”，与“正在优雅退出”打架。
- worker 在退出前 `heartbeat.stop()`（`main.ts:289`），心跳最迟 5s 内停止更新（`WORKER_BEAT_INTERVAL_MS = 5000`，`heartbeat.ts:28`）。`worker-healthcheck.mjs:17` 的阈值是 60s，**恰好够**——但这是巧合，不是设计。若将来把 `stop_grace_period` 调到 60s 以上，健康检查会在进程退出前就先判死。
- 方案没提这两个阈值之间的关系，部署参数一改就出问题。

**建议**

- 方案增加「**停机与健康检查的耦合**」一节，把三个数字写成显式约束：`stop_grace_period` > 进程自身兜底（web 10s）> 健康检查暴露窗口。worker 侧：`stop_grace_period` 必须 < `DSH_REVIEW_WORKER_STALE_MS`（默认 60000，`worker-healthcheck.mjs:17`），**与第二节的 300s 需求直接冲突**——这是方案必须正面处理的矛盾，不能回避。
- 建议停机前先让 `/api/health` 返回 503（主动摘流），再 `server.close()`。这样编排器/nginx 有据可依地停止转发，而不是靠连接被拒。

---

## 六、systemd 那套部署没有对齐

**问题部分**：方案只谈 compose（`proposal.md:17`），但 `deploy/` 下有完整的 systemd 单元集（`dsh-review-web.service`、`dsh-review-worker.service`、backup、discovery-drip）。

**后果**

- `deploy/dsh-review-worker.service:17` 是 `Restart=always` + `RestartSec=5`。systemd 默认 `KillSignal=SIGTERM`、`TimeoutStopSec=90s`。**systemd 与 compose 两套编排的停机预算完全不同**（90s vs 30s），方案只调 compose 的话，systemd 部署路径的 worker 行为仍是旧的、且未被验证。
- `dsh-review-worker.service:8` 有 `After=dsh-review-web.service`，但**没有停机依赖**（systemd 的 `After` 默认只影响启动，除非配 `Conflicts`/`BindsTo` 之类）。同样存在 web 先死、worker 还在写库的窗口。
- 方案完全没提这套，等于宣告“本次改造只覆盖容器部署”。若生产实际跑的是 systemd，那方案**没有交付任何东西**。

**建议**

- 方案必须**明确本次覆盖哪种部署形态**。若要都覆盖，补 systemd 侧：worker 的 `TimeoutStopSec` 与 SIGTERM 语义对齐，并说明 `Restart=always` 在**主动停机**（`systemctl stop`）时不会误拉起（systemd 会区分，但要写明已验证）。
- 加上“停机顺序”在两个编排下的做法，避免两套配置漂移（这正是 `default.conf:1-4` 注释里反复强调的教训）。

---

## 七、方案缺少运维验收标准

**问题部分**：`proposal.md:22-25`「预期：重启不丢请求 / 部署更平滑」

**后果**

- 这两条**没有可测判据**。“不丢请求”怎么量？丢多少算失败？“更平滑”如何验收？SRE 无法据此刻画 on-call 检查单，也无法事后复盘。
- 方案没有给出**任何日志/观测锚点**，而本仓库恰恰在这些地方准备好了：`server.mjs:44` 的 `server_shutdown_start`、`server.mjs:97` 的 `server_shutdown_done`、`server.mjs:104` 的 `server_shutdown_timeout`（warn）。这三条正好就是停机是否优雅的直接证据，方案一个字没引。

**建议**

- 把预期改成可验证条目，至少：
  - 部署脚本执行 N 次，`grep server_shutdown_timeout` 命中数为 **0**；`server_shutdown_start` 与 `server_shutdown_done` **成对出现**（且 done 的时间戳 - start < 10s）。
  - 部署期间用一个持续打请求的探针（如 `curl` 循环打 `/api/health` 和一个读接口），统计非 2xx 比例与连接错误数，给出**阈值**（例如 0 连接错误、5xx < 0.1%）。
  - worker 侧：部署后检查 `background_tasks` 中 `attempt_count` 因 SIGKILL 增长的任务数，确认是否符合“可接受重跑”的声明。
- 上线前给这三条日志加一句“运维 grep 指引”到 `docs/`，否则下次事故还是没人知道去哪看。

---

## 附：按严重度排序的整改清单

| # | 问题 | 位置 | 严重度 |
|---|---|---|---|
| 1 | `sleep(5000)` + `process.exit(0)` 会砍断在途请求，且掩盖失败；会退化已有的正确实现 | `proposal.md:8-13` vs `server.mjs:95-113` | **阻断** |
| 2 | worker 完全未覆盖；`stop_grace_period: 30s` < 单任务上限 300s，必然 SIGKILL 打断任务 | `proposal.md:5`、`compose.yaml:98`、`constants.ts:50` | **阻断** |
| 3 | 停机与健康检查阈值未对齐（worker: grace < 60s stale；web: 停机期 healthy→拒绝连接） | `worker-healthcheck.mjs:17`、`web.Dockerfile:73` | 高 |
| 4 | nginx 未先摘流，停机窗口必出 502；`resolver valid=10s` 未纳入方案 | `compose.yaml:131`、`nginx.conf:40` | 高 |
| 5 | `stop_grace_period: 15s` 是既有配置，方案误报为新增；worker 的 30s 与 15s 冲突 | `proposal.md:17`、`compose.yaml:75/98` | 中 |
| 6 | SQLite 收尾/WAL checkpoint 无约定；web 侧无 `closeDb()` | `main.ts:296`、`server.mjs`（缺） | 中 |
| 7 | systemd 部署形态未覆盖，两套编排预算不一致 | `deploy/*.service` | 中 |
| 8 | 验收标准不可测、未引用现成停机日志 | `proposal.md:22-25`、`server.mjs:44/97/104` | 中 |