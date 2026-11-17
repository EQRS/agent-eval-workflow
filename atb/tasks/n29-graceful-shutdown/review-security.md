# 安全评审结论 · n29「服务优雅停机改造」方案

评审对象：tasks/n29-graceful-shutdown/environment/proposal.md（25 行）
评审视角：安全工程师（仅安全面：可用性→鉴权绕过、失败关机降级、崩溃安全/数据完整性、凭证暴露、取证完整性）

## 摘要（前 5 条最重要）

1. 【严重】关机超时路径 `process.exitCode = 0` 而非瞬时退出 → 部署编排超时后 SIGKILL 落在**收尾中途**，把"优雅停机"降级为数据损坏风险 — 依据：server.mjs:109-112（10s 兜底只设 exitCode，进程仅在裸事件循环时才自然退出；worker 同款缺陷 src/worker/main.ts:248-256 的 stop() 只清 `running` 标志，从不 exit）；proposal.md:9-15 只写 `process.exit(0)`，未定义任何"到点强制结束"的安全边界 — 建议：兜底改 `try{closeDb()}finally{process.exit(code)}`，退出前显式 `server.closeAllConnections()`，并把该时限与 stop_grace_period 的关系写进方案（见第 2 条）。

2. 【高】方案给的信号处理骨架与本仓既有实现**不一致且更弱**：`sleep(5000)+exit(0)` 是"固定等 5 秒再无条件退出"，等于给在途请求设了硬截止；本仓现有实现是 `server.close()` 回调 + 10s 兜底（server.mjs:100-113），两者混用会出现"双份 SIGTERM 处理器"的竞态 — 依据：server.mjs:98-115 已注册 SIGTERM/SIGINT；proposal.md:10-14 会新增第二组监听器，Node 会**两个都跑**（无 once 语义冲突保护，仅靠彼此各自的 closing/running 标志）— 建议：方案应写"改造既有 shutdown()"，而不是给一段新骨架；删掉 sleep 固定等待，改为「停接新连接 → 等在途完成 → 兜底强杀」三段式。

3. 【高】`stop_grace_period: 15s` 与 web 自身 10s 兜底、worker 的 30s 之间的预算关系未定义 → 任一环节超时就退化为 SIGKILL，而 SIGKILL 掉的正是"已写完文件、未提交数据库"的中间态 — 依据：docker/compose.yaml:69（web 15s）、docker/compose.yaml:94（worker 30s）、server.mjs:109-112（10s）、src/server/platform/file-store.ts:96-124（先 rename 落地、后回读校验，两步之间被切断即留下无主文件）— 建议：方案必须给出「各层时限 < 上层时限」的预算表，并声明 SIGKILL 是**不可接受**的收尾方式（而非"容器 stop 超时"的备注）。

4. 【高】SIGKILL 会造成**可被利用的孤儿媒体对象**，方案完全未提 — 依据：file-store.ts:106-107 `writeFile(tmp,...)`→`rename(tmp,target)` 与 media 的登记顺序之间无补偿；`*.tmp-<pid>-<ts>` 只在异常路径清理（file-store.ts:108-111），进程被强杀时既不清理也不登记，无人回收；更关键的是 `rename` 成功但 DB 事务未提交时，文件已可被按 key 直接读取 — 建议：方案增加"启动时清扫孤儿 .tmp- 与未登记对象"的步骤；或明确写入文件必须在 DB 提交**之后**（当前顺序相反）。

5. 【中】停机期间的错误语义未定义 → 反代侧会把 5xx 当"后端异常"而非"维护中"，且方案未处理在途连接的 RST — 依据：nginx 对 upstream 断连不做重试配置，docker/nginx/snippets/proxy-common.conf:39-41 只有超时（connect 5s / send 60s / read 60s），无 `proxy_next_upstream`；`server.close()` 只停接新连接，已建立但空闲的 keepalive 连接（proxy-common.conf:34-38 明确启用了 upstream keepalive 16）不会被立即关闭 — 建议：shutdown 中显式 `server.closeIdleConnections()`；停机窗口应由编排层先摘流（nginx `down`/健康检查失败）再发 SIGTERM，方案需写出这个顺序。

## 详细发现

### 一、可用性缺陷是否转化为鉴权绕过
**结论：未发现直接绕过，但存在一条"时间窗"风险，需显式排除。**

- 频控计数写在库里、且为单语句原子自增（src/server/platform/rate-limit.ts:14-35、172-188）。进程被杀不会重置计数，所以"靠重启刷额度"这条路**不通**——这点方案可以放心。
- 但 rate-limit.ts:29-34 记录了一条行为：**被拒请求也会累加 points**，而 `expire` 保持首写值。频繁重启不改变这一点，不构成新问题。
- 真正需要方案回答的是：**在途请求被杀后，已消耗的额度是否退还**。看 rate-limit.ts:284-293 的 `refundQuietly`——它是"业务失败后善后"。若进程在业务写入前被 SIGKILL，ticket 既没退也没用，用户白扣额度；这不是安全漏洞，但**是可被观察到的侧信道**（攻击者可用"上线窗口内反复压请求"探知部署时刻）。建议方案写明"停机窗口内 429/529 与正常 429 的区分"，避免监控把两者混为一谈。

### 二、失败关机是否降级为不安全状态
**这是本方案最该补的一节，当前完全缺失。**

1. **降级粒度不明确**：server.mjs:104-107 的成功路径只设 `process.exitCode = 0`，前提是"事件循环自然枯竭"。但 Next 会在进程内持有内部定时器/池（Next 16 自建 server 场景），一旦这些句柄不释放，进程会**卡在 10s 兜底之后**——编排器到 `stop_grace_period: 15s`（compose.yaml:69）直接 SIGKILL。也就是说：**兜底"失败"时没有任何机制保证进程真的退出**，方案必须把"强制结束"写成显式动作。
2. **数据库收尾缺失**：web 侧**从不调用 `closeDb()`**（全仓只有 worker main.ts:284 与 migrate-cli.ts:20/24 调用）。WAL 模式下，未 close 的连接由内核在进程退出时释放，一般不会损坏；但 `synchronous = FULL`（db.ts:54）下的最后一次 fsync 是否完成，取决于进程是自然退出还是被 SIGKILL。方案应明确："优雅停机的收益是**保证最后一次提交落盘**"，并为此在 shutdown 里加 `closeDb()`。
3. **worker 侧完全没有超时**：src/worker/main.ts:6 的注释承诺"SIGTERM 优雅退出（不再领新任务，等在跑的任务结束或租约到期）"，但 248-256 的 `stop()` 只翻标志，运行中的 handler 可能跑到 `TASK_DEADLINE_MS = 300s`（src/server/modules/tasks/constants.ts:45）；而 compose 给 worker 的 `stop_grace_period` 只有 30s（compose.yaml:94）。**必然 SIGKILL**。好在任务队列有租约兜底：`recoverExpired` 会在 `LEASE_DURATION_MS = 120s`（constants.ts:19）后回收，且 `finish`/`renew` 都是 CAS（service.ts:243-266、278-382、398-444）。**但这正好暴露真正的安全问题**——见第三节。

### 三、崩溃安全与静默的权限不一致（本方案未覆盖的最大安全面）
**任务被 SIGKILL 后由新进程"续跑"，而续跑者没有"上一次执行进度"的语义保证。**

- 证据：serve.ts 的 CAS 只校验 `execution_token + running + lease 未过期`（src/server/modules/tasks/service.ts:243-255、278-382）。租约到期回收后，**同一 payload 的任务会被另一个 worker 实例从头再跑一遍**。
- 对 `plugin_discovery` 这类外部动作是安全的（幂等键 + dedup），但对 `review_publish`/`comment_publish` 呢？handlers.ts:191-207 依赖 `publishReview` 在事务内重核状态并返回 `already_published` 幂等命中——**这一点是全仓支撑"重复执行无害"的承重墙**。若某次改造（含本方案的"启动即清扫/重排"）让发布路径绕过该事务内重核，**重复执行就会变成重复公开**。
- 因此本方案**不得**引入任何"启动时把 running 任务直接重置为 pending"的粗暴回收——那会绕过 CAS 语义、把租约到期变成"任何人可立即重放"。
- 建议：方案明写"回收只准走 `recoverExpired`（租约到期 + CAS），禁止启动时批量重置状态"。

### 四、凭证与配置暴露面
- 方案第 3 节只加 `stop_grace_period`，不触碰 env。**但有一个既有问题会被停机流程放大**：`DSH_REVIEW_TRUST_PROXY: '1'`（compose.yaml:59）配合 nginx 的 `X-Forwarded-For` 覆盖（proxy-common.conf:16-23）。
  - 若为排查停机而在 web 上临时"发布 127.0.0.1:3101:3000"（compose.yaml:63 的注释正是这么建议的），**该端口不经过 nginx，应用仍信任 XFF** → 直连本端口的人可完全自选 `client_ip`，于是**频控、审计日志、滥用追踪全部被伪造**。
  - 这不是方案引入的 Bug，但方案里"部署配置"一节是修改 compose 的正当位置，安全侧要求在这里补一句硬约束：**调试端口打开时必须同时置 `DSH_REVIEW_TRUST_PROXY=0`**。
- 凭证本身（`SESSION_SECRET`、`GITHUB_TOKEN`）由 compose `env_file` 运行时注入（web.Dockerfile:67-69），不烘进镜像；停机改造**不要**新增"退出时打印配置摘要"之类日志，否则会把 env 带进 journal/日志驱动（compose.yaml:70-74）。

### 五、取证完整性（安全事件复盘的前提）
- nginx 侧请求 id 透传是 `proxy_set_header X-Request-Id $http_x_request_id`（proxy-common.conf:32）——**客户端可控**。server.mjs:54-60 直接沿用入站 id，不做格式/长度校验就把 id 写进响应头（server.mjs:60）。
- 停机期间正是"用户报障、运维回捞日志"的高峰。若方案把停机日志（`server_shutdown_start/done/timeout`，server.mjs:103-110）与请求日志混在同一输出，攻击者可用**超长/含换行的 x-request-id** 伪造整行 JSON 日志（writeLine 只做 `JSON.stringify`，字段本身会被转义——**此处实际安全，JSON.stringify 会转义换行**；但 id 未校验长度，日志注入面虽被转义封闭，**日志膨胀/污染**仍存在）。
- 结论：方案要求的"停机日志"应使用**固定 schema、不含任何请求头内容**，并建议后续给 x-request-id 加 UUID 形状校验（`server.mjs:54`）——这条属可另开切片，但方案应至少不要去"增强"它。

### 六、"预期"一节的验证缺口
方案第 24-25 行写"重启不丢请求"，但没有任何验收手段。安全侧要求至少能证伪以下三条（否则等于没做）：
1. **在途写请求不产生半截状态**：全站写入口都走 `withTransaction`（src/server/platform/tx.ts:42-52，同步事务、失败自动 rollback）。这是好事——但 **文件写入在事务外**（file-store.ts:96-124 是异步 fs，不可能在同步事务里）。所以"不丢请求"必须限定为"**不产生数据库半截写入**"，文件侧做不到。方案必须承认这条边界。
2. **不产生孤儿文件**：停机后 `.data/media` 下应无 `*.tmp-*` 残留。这条可直接作为验收断言。
3. **权限不变**：停机/重启前后，同一用户组的 `assertCanWrite` 行为一致（context.ts 层无进程内缓存——需要开发同学确认，安全侧未发现缓存层）。；

## 建议的最小改造清单（安全视角）
1. 把 shutdown 收敛成**唯一一组**处理器，替换 server.mjs:98-115 既有实现，删除 proposal.md:9-15 的独立骨架。
2. 兜底路径必须 `process.exit(非0)`，且退出前 `closeDb()`（web 侧当前完全缺失）。
3. 显式 `server.closeIdleConnections()`，解决 keepalive 空闲连接拖到兜底超时的问题。
4. 给出时限预算表：`server 内部兜底 < compose stop_grace_period < 编排/反代摘流窗口`，并说明 worker 侧 300s 任务上限与 30s 停机窗口的冲突如何处理（建议：优先 `closeDb()` 前先让当前任务**完成写回或主动放弃**，而不是硬杀）。
5. 明确禁止"启动时批量重置 running 任务"，回收只走 `recoverExpired`。
6. 补两条验收断言：无 `*.tmp-*` 残留；停机后 `/api/health` 与 `selfCheck` 仍 ok。
