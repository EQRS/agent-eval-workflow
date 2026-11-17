# N29「服务优雅停机改造」方案 · 安全工程师评审

> 评审对象：`atb/tasks/n29-graceful-shutdown/environment/proposal.md`（25 行）
> 评审身份：安全工程师（边界：停机窗口是否制造鉴权绕过 / 崩溃安全 / 文件与媒体一致性 / 会话与凭据 / 取证完整性）
> 事实基线：`server.mjs`、`src/worker/main.ts`、`src/server/platform/{db,file-store,session-cookie,client-ip,request-id,heartbeat}.ts`、`src/server/modules/{tasks/service,tasks/constants,identity/service}.ts`、`src/app/api/health/route.ts`、`docker/compose.yaml`、`docker/nginx/snippets/proxy-common.conf`、`deploy/dsh-review-web.service`

## 0. 结论

**不建议按现方案实施。** 从安全面看，方案有三处「把正常运行状态换成不安全状态」的风险：

1. **停机期间健康检查仍返回 200**（§3）→ 负载均衡继续派流量，杀进程时在途连接被 RST，**重试风暴打在正在退出的进程上**；而 `/api/health` 是公网可读的，方案也没定义「draining」这个第三方可见的状态语义。
2. **SIGKILL 是必然结局，而它不是"少等一会儿"**（§4）→ 文件已 `rename` 落地、数据库事务未提交的中间态会留下**无主媒体文件**（`file-store.ts:106-107`）；这类残留可以长期按 key 直接读取，是**真实的越权读取面**。
3. **会话与凭据面的默认假设没有写进方案**（§5/§7）→ 停机会话失效被新的 SIGTERM 处理器逻辑替换后，"退出即撤销会话"很容易被顺手误伤；同时 `DSH_REVIEW_TRUST_PROXY=1` 与调试端口共存时 IP 可伪造，直接影响审计与频控。

**方案原文只有 25 行，其中第 9–15 行的信号处理骨架在本仓是「已经实现过一遍的」——见 §1，这次改造真正的安全风险全部来自"改造时怎么改"，而方案对这部分一个字都没写。**

---

## 1. 方案要解决的问题在本仓已经实现过一半，且现有实现比方案更强 —— proposal 第 8–15 行

**① 问题**：方案给出一段新的 SIGTERM 骨架（`server.close(); await sleep(5000); process.exit(0)`）。但 `server.mjs:98-115` **已经**实现了 shutdown：三阶段（`server.close()` → `server_shutdown_done` → 10 s 兜底 `server_shutdown_timeout`），并用 `closing` 标志做幂等。worker 侧也有（`src/worker/main.ts:248-256`）。

**② 后果**：
- 如果照着方案新增一组监听器，Node 会**同时执行两组 `process.on('SIGTERM')``——它们各自有 `closing`/`running` 标志，但**互相看不见对方**。结果是：现有实现里"关掉 DB、记 `server_shutdown_done`"的收尾与新的 `process.exit(0)` **竞态**，谁先谁后不确定。
- 最坏的那条路径是：新处理器 5 s 到点 `process.exit(0)` **先执行**，而现有实现的事务/连接收尾**没跑完** → 进程以退出码 0 结束，运维与编排器看到的是"优雅退出成功"，实际是**中途硬切**。**这是一种安全相关的失败：故障被伪装成成功。**
- 方案没有任何"替换而非新增"的表述，也没有指出 `server.mjs` 已注册过这两个信号——按字面实现就会双注册。

**③ 建议**：
- 方案必须写明：**改 `server.mjs:98-115` 的那一个函数，不新增监听器**。
- 在 `server.mjs:100-102` 的 `if (closing) return;` 之上补一条 `process.once` 语义（或显式 `process.removeAllListeners('SIGTERM')`），并在方案里把"信号处理器唯一性"写成不变量。

---

## 2. `sleep(5000)` 不是一个安全边界，而是一个竞猜 —— proposal 第 10–18 行

**① 问题**：`server.close()` + 固定 5 s + `process.exit(0)`。

**② 后果**：
- `server.close()` **只停止接受新连接**，不等待在途请求；方案用固定 sleep 盖住"等待"，等于**赌所有请求都在 5 s 内完成**。本仓最慢的合法路径远超 5 s：`db.ts:56` 的 `busy_timeout = 5000` 单独一项就吃满 5 s，加上同步查询与文件 `sha256` 回读校验（`file-store.ts:102-121`，整块 buffer 读写 + 两次哈希）后，一次大附件上传轻易越过。
- 越过的那一次，`process.exit(0)` 会在**请求处理中途**把进程杀掉。于是：**服务端已经写入的部分数据、已经 rename 落地的文件，与客户端看到的"连接被重置"** 三者互不一致。
- 安全视角特别关注这一条：**客户端会重试**。重试打在同一份数据上，是否幂等取决于业务层；而方案没有为"停机窗口"定义任何幂等语义（对比 `content_submissions(submitted_by, request_key)` 这套去重键，见 `src/server/platform/request-id.ts:6-15` 的说明）。

**③ 建议**：
- 改成**等待活跃连接归零**（`server.close(cb)` 的回调在最后一个连接关闭时触发，`server.mjs:104-107` 已经在用），配合一个**明确的超时上限**；超时上限要 `<` 外层的 `stop_grace_period`。
- 超时到点必须 `process.exit(非 0)`，让编排器/日志知道"这次不是干净退出"——**不要用 0**。
- 方案里补一句：<b>停机窗口内的写请求不保证幂等，客户端重试需依赖业务层 `request_key`</b>。

---

## 3. 没有 drain 状态：`/api/health` 在停机期间仍返回 200，公网可读且无「维护中」语义 —— proposal 全篇

**① 问题**：方案没有任何"停止接流量"的步骤，只 sleep 5 s 后退出。

**② 后果**：
- 停机期间 `/api/health` 依旧 200（`src/app/api/health/route.ts:30-58` 只看库与磁盘），`docker/web.Dockerfile:76-77` 的 HEALTHCHECK 也依旧通过 → 负载均衡/编排器**继续把流量打进正在退出的进程**，只是把 RST 分散到更多用户身上。
- 而且这个端点**公网可读**（`route.ts:1-14` 的契约写得很清楚：body 只放 `status/checked_at/schema_version`，细节要 `x-health-token`），意味着**任何人都可以观察服务的生命周期状态**。方案若按常规做法加一个 `draining` 字段，就等于把"部署窗口"变成一个外部可读的时间信号——**攻击者可据此选择在部署期间压测/投放**。
- 反代侧也没有摘流动作：nginx 到 web 的 upstream keepalive 是 16 条（`proxy-common.conf:34-38`），`server.close()` 不会立即关闭这些**已建立的空闲连接**；方案 5 s 的等待期内它们仍会被复用。

**③ 建议**：
- drain 语义**只对内**：用一个独立内部端点（或 `x-health-token` 保护的 `?detail=1` 字段）表达"我要退了"，**不要**把 drain 状态加进公网 body（`route.ts:10-12` 的现有边界别破）。
- 先让 nginx 摘流（upstream down / 健康检查失败），**再**发 SIGTERM；顺序必须写进方案：**摘流 → 等在途 → 退出**。
- shutdown 里显式 `server.closeIdleConnections()`（Node 18+），否则空闲 keepalive 连接会把退出拖到兜底时限。

---

## 4. 必然 SIGKILL，而 SIGKILL 会留下**可越权读取**的孤儿媒体 —— proposal 第 20–21 行

**① 问题**：方案只在 compose 里设 `stop_grace_period: 15s`，没有回答"如果 15 s 内没退干净会怎样"。

**② 后果**：
- 本仓**必然**触发 SIGKILL：
  - worker 侧：`stop()` 只翻 `running` 标志（`worker/main.ts:248-256`），从不 `process.exit`；在跑的任务最多可以跑到 `TASK_DEADLINE_MS = 300s`（`tasks/constants.ts:45`），而 compose 给 worker 的停机窗口只有 30s（`compose.yaml:94`）。**10 倍差距**。
  - web 侧：兜底路径只设 `process.exitCode = 0`（`server.mjs:109-112`），**不强制结束**；若 Next 持有内部句柄，进程会拖到 15 s 后被 SIGKILL。
- 被 SIGKILL 的那一刻正在做什么，决定了留下什么：
  - `file-store.ts:106-107`：`writeFile(tmp)` 之后 `rename(tmp, target)` ——`rename` 是原子的，所以**不会留下半截文件**（这点方案可以放心）；但会留下 `*.tmp-<pid>-<ts>`（:101 的命名），而清理只在 `catch` 里做（:108-111），**进程被强杀时没有任何清理路径**。
  - 更关键的是 **`rename` 成功、数据库事务还没提交** 的顺序。文件一旦落到最终 key，就**不经过任何可见性判定**地存在于磁盘上，而 `/media/{attachmentId}/{variantKey}` 的可见性是由应用按会话与内容状态判的（`site-locations.conf:29-36` 明确写了"绝不能改成直接读盘"）。**"文件已落地、行未登记"意味着这份文件脱离可见性体系**——它能被按路径探测到，或至少会在后续"同一 key 覆盖写"时产生**内容与元数据不一致**。
- 还有一条安全面：**任务被 SIGKILL 后的恢复**。租约到期由 `recoverExpired` 回收（`tasks/service.ts:398-467`），CAS 校验 `execution_token + running + lease_expires_at > now`（:14），**这是承重墙**。方案第 3 节是改部署配置的正当位置，但任何"启动时把 running 批量重置为 queued"的补充方案都会**绕过 CAS**，让过期租约立刻可被任意 worker 重放——如果目标任务是 `review_publish`，就是**重复公开**。

**③ 建议**：
- 兜底路径必须 `process.exit(非 0)`，且在退出前显式 `closeDb()`（web 侧**当前从不 close**，全仓只有 worker `main.ts:283-284` 与 `migrate-cli` 会 close）——`synchronous = FULL`（`db.ts:54`）下最后一次提交是否落盘，取决于进程是自然退出还是被 SIGKILL。
- 在方案里明确 **"文件写入必须发生在数据库事务提交之后"**，或补一条**启动时的孤儿清扫**（扫 `*.tmp-*` 与"磁盘有文件但库里没有对应行"的对象），并给出去重/保留策略。这条应作为安全验收项，不只是运维清理。
- worker 侧必须补**主动退出**：`shouldContinue()` 为 false 之后，等待当前 `execute` 结束（或主动放弃），然后 `process.exit`。**不允许**把 `TASK_DEADLINE_MS = 300s` 留在 30s 的停机窗口里赌。
- 方案里加一条硬性约束：**禁止"启动时批量重置 running"**，回收只走 `recoverExpired`（CAS）。

---

## 5. 会话面：没有 `Set-Cookie` 缓存，被杀的连接不会留下"半登录"——但方案没说这一点，改造时容易拆掉它 —— proposal 全篇

**① 问题**：方案完全没有讨论停机期间的会话与 Cookie 语义。

**② 后果（正面结论也值得写进方案，因为它是**误改的陷阱**）**：
- 核实结论：**不存在"Cookie 已下发、数据库会话未落库"的窗口**。`resolveSession`（`identity/service.ts:245-276`）**每次从库里读**，用户、角色、`account_status` **不缓存在进程内**（:262-267 的注释明确写了"角色与账户状态每次从库里读，不缓存"）。所以重启不产生"权限残留"。
- 但反过来：**如果为了"优雅停机"给会话加进程内缓存**（这是很常见的提速手段），就会立刻制造一条新风险——用户被管理员封禁（`identity/admin.ts:569` 批量 `UPDATE user_sessions SET revoked_at`）、或退出登录后，缓存仍会在停机窗口内继续放行。方案要在"不要做什么"里**显式禁止**这条改造。
- 另外，`DSH_REVIEW_INSECURE_COOKIES=1`（`session-cookie.ts:15-21`）会让 Cookie 在明文 HTTP 下可用；停机/重启期间若有人为了排查临时加上它再忘删，会话凭据就在链路上裸奔。方案第 3 节是改部署配置的位置，安全侧要求在这里补一句"停机相关临时开关必须在方案里登记回收"。

**③ 建议**：在方案里写两行：**（a）会话与权限不引入任何进程内缓存**（保持 `identity/service.ts:262-267` 的现有契约）；**（b）停机相关的临时环境变量（`DSH_REVIEW_INSECURE_COOKIES`、调试端口）必须列入"随部署结束回收"的清单**。

---

## 6. 停机窗口是**攻击面的高价值时段**：错误码、限流与日志的混叠 —— proposal 全篇

**① 问题**：方案假定"重启不丢请求"，但没有定义**停机期间对外返回什么**。

**② 后果**：
- **连接被 RST vs 明确的 503**：RST 在客户端看来是"网络错误"，会触发**自动重试**（浏览器、SDK、监控探针都这样）；而 503 + `Retry-After` 是"请稍后再来"。方案让前者成为默认，等于在每次部署时制造一轮**自带放大的重试风暴**，并且这些重试全部落在刚重启、**缓存冷、连接池空**的新进程上。
- **频率限制的语义混叠**：`rate-limit.ts:29-34` 记录了一条既有行为——**被拒请求也会累加 points**。停机窗口内被 RST 的请求**没有走到频控**，所以不会消耗额度，这点是好的；但反过来，重试风暴会在重启后**集中消耗额度**，把正常的用户打进 429。方案没有区分"部署导致的 429"与"滥用导致的 429"，监控会把两者混为一谈。
- **取证**：`server.mjs:54-60` 沿用入站 `x-request-id`（`request-id.ts:36-48` 只校验字符集与长度，**不校验来源**），而停机日志（`server_shutdown_*`，`server.mjs:103-110`）与请求日志共用同一输出流。停机正是"用户报障、运维捞日志"的高峰，方案新增的停机日志**必须是固定 schema、不含任何请求头内容**，否则攻击者可用可控的 `x-request-id` 把可巡检的日志行与停机记录混在一起（`JSON.stringify` 会转义换行，所以**不是注入**，但是**污染与检索干扰**）。

**③ 建议**：
- 停机期间对外统一 **503 + `Retry-After`**（明确区别于异常 5xx），并在方案里写明它**不计入错误率 SLI**。
- 停机日志固定字段（`signal / phase / active_connections / elapsed_ms`），**不带任何请求头**。
- 频控侧在方案里补一句：重启后的集中重试属于已知放大源，需在 SLO 里单独标注，不要当成滥用。

---

## 7. 调试端口与 `TRUST_PROXY`：方案改的是 compose，正是该补这条硬约束的位置 —— proposal 第 20–21 行

**① 问题**：方案第 3 节改 `docker/compose.yaml`，但只加了一个 `stop_grace_period`。

**② 后果**：
- `compose.yaml:57-59` 注释本身就建议"排查时临时加 `127.0.0.1:3101:3000`"，同时 `DSH_REVIEW_TRUST_PROXY: '1'` 是常开的。**一旦把 web 端口发布出来，客户端就可以自己写 `X-Real-IP`/`X-Forwarded-For`**（`client-ip.mjs:54-67` 在 `trustProxy` 为真时**无条件下**信任这两个头），于是 `client_ip` 可被任意伪造 → **审计日志失真、按 IP 的频控失效、滥用追踪归零**。
- 方案在改 compose 时不写这条，等于**把一次配置改动变成了一个新的越权面**：以后任何人为了排障加端口，都不会想起要先关 `TRUST_PROXY`。

**③ 建议**：在方案的"部署配置"一节写死：**发布任何 web 端口时必须同时置 `DSH_REVIEW_TRUST_PROXY=0`**；更稳妥的做法是把 `trustProxy` 与"是否发布端口"绑定成同一个 compose 变量（二选一），从结构上消除这个组合。

---

## 8. worker 心跳与"存活"判定：停机期间会被误读成"死了"，从而触发错误的自动恢复 —— proposal 全篇

**① 问题**：方案只覆盖 web 的信号处理，没有定义 worker 停机期间的**心跳语义**。

**② 后果**：
- worker 心跳靠 `startHeartbeat` 定时写（`heartbeat.ts:132-151`），`WORKER_STALE_MS = 30s` / `DEAD_MS = 120s`（:20-23），`?worker=1` 时 health 直接 503（`health.ts:127-133`）。
- worker 收到 SIGTERM 后 `stop()` 只翻标志（`worker/main.ts:248-256`），**心跳仍每 5 s 写一次**，直到进程真正退出。这看起来是"好的"，但它**掩盖了正在退出这件事**——编排器与监控完全看不出它要退了。
- 更危险的是**重启后的竞态**：新 worker 用 `beginRun` 重置 `started_at`（`heartbeat.ts:77-95`），而旧进程若尚未退出（SIGKILL 前那 30 s），**两个 worker 会同时领任务**。任务侧的 CAS（`service.ts:14`）保证了不会重复写回，这一点是安全的；但**心跳行会被两个进程互相覆盖**，运维从心跳里读到的"存活时长"是不可信的——方案若据此做"何时该杀"的判断，就会杀错。

**③ 建议**：worker 在收到 SIGTERM 后应**主动把心跳标成 draining**（`detail` 里加 `stopping: true`），并**停止领新任务**（`shouldContinue` 已经是这个语义，写清楚即可）；同时方案要求**退出前停心跳**（`heartbeat.stop()` 已在 `main.ts:283` 的 finally 里，但必须在 `process.exit` 之**前**执行——这条要写死）。

---

## 9. 验收：方案第 24–25 行「重启不丢请求」无法被证伪 —— proposal 第 23–25 行

**① 问题**：预期只有两句，没有任何可测断言。

**② 后果**：无法验证的"不丢请求"等于没有承诺；更糟的是它会**掩盖失败**——因为真正的失败形态（§1 退出码 0 的假成功、§4 的孤儿文件、§2 的半截请求）都发生在"看起来成功了"的路径上。

**③ 建议**（安全侧要求的最小证伪集）：
1. **不假成功**：注入一个超过停机窗口的在途请求，断言进程以**非 0** 退出码结束，日志里有 `server_shutdown_timeout`。
2. **不留孤儿**：停机后 `.data/media` 下无 `*.tmp-*`；随机抽 10 个磁盘对象，断言都能在库里找到对应行。
3. **不重复公开**：在 `review_publish` 执行中途发 SIGTERM，重启后断言该评测**只公开一次**（`operation_records` 的 `operation_key` 去重仍然成立）。
4. **不越权**：停机前后同一用户的 `assertCanWrite` 行为一致；被封禁用户的会话在重启后立刻失效（验证 §5 的"不缓存"契约没被破坏）。
5. **不伪造 IP**：开启调试端口且未关 `TRUST_PROXY` 时，构造 `X-Real-IP` 头，断言 `client_ip` 被伪造（把这条写成**已知风险**，而不是让它悄悄通过）。

## 10. 一句话给作者

方案 25 行里，**第 8–15 行是仓库里已经实现过、而且现有实现更强的一段**（`server.mjs:98-115` 三阶段 + 兜底），真正需要写的是**它没有写的部分**：`exit` 的退出码与 `closeDb()`、文件与数据库的提交顺序、worker 的主动退出、drain 与反代摘流的顺序、以及"禁止启动时批量重置任务"这条 CAS 承重墙。**"停机"在安全上的意义，不是"少等一会儿"，而是"不要在收尾中途被切断"。**
