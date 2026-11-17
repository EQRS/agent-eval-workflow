# n17-data-leak · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0001
> 模型 deepseek-chat · 7769 字 · 18 次工具调用

---

证据已经齐了。下面是评审。

---

# SRE（运维）评审：基于访问日志的用户行为分析

评审对象：`atb/tasks/n17-data-leak/environment/proposal.md`（全文 38 行）
评审角度：**只看运维面**（日志落点与容量 / 保留期 / 隐私批处理 / 备份 / 可观测性退化 / 故障处置）。产品与合规问题只在与运维后果直接相关时提到。

**先说一句最重要的**：这份方案在运维视角下**根本跑不起来**，因为第 3 节「日志写入 JSONL 文件，保留 90 天」与现状不符；即使实现了，也会**把唯一的排障通道（journald）弄得更难用**，还会引入一个**无上限的磁盘增长源**。

---

## 一、事实基线（我实际读到的，不是猜的）

| 事实 | 出处 |
|---|---|
| 访问日志当前由 `server.mjs` 写入 **stdout/stderr**，不写文件 | `server.mjs:43-47` |
| 生产部署把 stdout **重定向进 journald（systemd）** | `deploy/dsh-review-web.service:27-28`（`StandardOutput=journal` / `SyslogIdentifier=dsh-review-web`） |
| Docker 部署则用 json-file，**上限 10m × 3 = 30m 封顶** | `docker/compose.yaml:72-75` |
| 现有访问日志字段已含 `client_ip` 与每请求 `request_id` | `server.mjs:96-103` |
| nginx 访问日志**已经记录了 referer 与 user_agent** | `docker/nginx/nginx.conf:20-25` |
| 健康检查有 **200MB 磁盘余量硬闸**，低于即判 fail→503 | `src/server/platform/health.ts:50` |
| 现有审计表被加了不可删、不可改触发器 | `migrations/0042_audit_immutable.sql:19-27` |
| `currentContext()` 未登录时返回**匿名上下文**（userId 为空） | `src/app/api/_lib/context.ts:20-25` |

---

## 二、问题清单（运维视角）

### 【P0-1】第 3 节「写入 JSONL 文件」在生产不存在这条链路 —— 日志的落点和保留期完全没定

- **哪部分有问题**：`proposal.md:2`（「现有 server.mjs 每请求一行访问日志」，暗示已写文件）与 `proposal.md:27`（「日志写入 JSONL 文件，保留 90 天」）。
- **后果**：`server.mjs:43-47` 的 `writeLine` 只做 `process.stdout.write` / `process.stderr.write`，**从不打开文件**。systemd 部署下日志进 journald（`dsh-review-web.service:27-28`）；compose 部署下进容器 json-file，**被 10m×3 截断**（`docker/compose.yaml:72-75`）。也就是说：① 「保留 90 天」现在**一天都保证不了**，日志由 journald `SystemMaxUse` 或 docker json-file 轮转决定；② 方案要的 JSONL 文件**没有任何代码会产生**，也没说写到哪个目录、属主是谁（容器内 UID 10001）、谁来轮转。运维实施时会在「日志到底在哪」这一步直接卡住。
- **建议**：先明确落点再谈分析。给三种可选形态并选定一种：(a) 保持 journald，用 `journalctl --since` 做回溯，方案改说「保留期由 journald 配置保证」；(b) 独占写 `/app/.data/logs/access.jsonl`（必须落在 `DSH_REVIEW_DATA_DIR` 内，否则违反 `docs/docker约定.md` §3「除 /root 外不留痕」），并明确属主 10001 与 logrotate 参数；(c) 直接复用 nginx access log。**不接受**「边说写文件、边假设已经写了」这种两头空的表述。

### 【P0-2】90 天保留 + 每请求加 user_agent/referer，磁盘没有任何容量测算，而站点只有 200MB 余量闸

- **哪部分有问题**：`proposal.md:19-21`（新增 `user_agent`、`referer` 字段）与 `proposal.md:28`（保留 90 天）、`proposal.md:36`（「都是文本，成本可控」）。
- **后果**：`health.ts:50` 的 `DEFAULT_MIN_FREE_MB = 200` —— **磁盘余量低于 200MB，`/api/health` 直接判 fail、全站 503**。`user_agent` 与 `referer` 是**长度无上限的客户端可控字段**（UA 常 150–300 字节，referer 可到 KB 级），加上 90 天保留，是本仓**第一个无上限、不清理的写入型增长源**。方案的「成本可控」只有一句定性判断，没有任何 QPS × 平均行长 × 86400 × 90 的估算。一次被刷（爬虫或压测）就能把这个余量吃掉，表现是**健康检查挂掉、容器被判死**，而不是「日志多了一点」。另外注意 `rate-limit.ts` 那边已经有一条 `clearExpiredByTimeout: false`（限流行永不过期），盘上已有常驻不清理的数据，余量比看起来更紧。
- **建议**：方案里必须给出**容量预算表**（估算行数与字节、90 天总量、占盘比），并硬性写明：① 单个字段截断上限（如 UA/referer 各截 256 字节，referer 只保留 origin+path）；② 总盘占用上限 + 超限行为（丢弃最高频/最低价值记录，而不是写满磁盘）；③ 把日志目录排除在健康检查的磁盘口径之外，或明确告警阈值与 200MB 闸的间距。**没有这三个数，这条不能上线。**

### 【P0-3】字段是无上限的客户端输入，直接 `JSON.stringify` 写行 —— 日志注入与解析破坏

- **哪部分有问题**：`proposal.md:22-23`（`user_agent: req.headers['user-agent']`、`referer: req.headers.referer` 原样进日志）。
- **后果**：现行 `writeLine` 用 `JSON.stringify`（`server.mjs:44`）虽能转义引号，但**换行与超长仍会破坏「一行一条」这条基本约定** —— 方案 §2「按 user_id 聚合还原路径」正依赖逐行解析。一条含 `\n` 或数 KB 的 UA 就能让下游按行切分的脚本错位；更严重的是，`referer` 可以带完整 URL，**query string 里常有 token、邮箱、重置密码链接**，这些会被永久写入 90 天日志。这与仓库已有的谨慎口径明显不一致：`0003_auth_media.sql:23-25` 明确「只存摘要，不存 state / 浏览器绑定原值」。
- **建议**：在**写入侧**（不是分析侧）做白名单化：`referer` 只存 `scheme://host/path`，**丢弃 query 与 fragment**；UA 截断到固定长度；所有字段过一遍「去控制字符」；并给出「敏感 query 参数名名单」的剥离逻辑。这条应写进方案，而不是留给实现随手处理。

### 【P0-4】把 `user_id` 写进访问日志，等于把「匿名可访问的日志通道」变成「身份关联库」，而日志没有访问控制与生命周期约定

- **哪部分有问题**：`proposal.md:14`（`user_id: ctx.userId`）与 `proposal.md:24`（按 user_id 聚合）。
- **后果**：日志一旦带上 `user_id` + `client_ip`（`server.mjs:96-103` 已有）+ UA + referer，就是一个**跨表的用户画像数据集**，但它存放在**最不受管控的地方**——journald / 容器 json-file（`dsh-review-web.service:27-28`、`docker/compose.yaml:72-75`），而这两个通道：① 没有库级别的不可篡改（对比 `0042_audit_immutable.sql:19-27` 刚给审计表加的 `no_delete`/`no_update` 触发器）；② **不随备份策略走**（`docker/compose.yaml:66-70` 备份只覆盖 data 与 backups 目录）；③ 没有留存删除流程。运维后果是：**用户要求删除/更正时，我们没有任何手段在日志里定位并抹掉他的行**——因为日志是追加的、轮转的、且没有稳定主键。这是真正会卡住运维动作的形状，不是理论风险。另外 `currentContext()` 未登录返回匿名（`src/app/api/_lib/context.ts:22`），`user_id` 会是空值，聚合口径需要明确「匿名如何处理」，否则「未登录用户全部并成一个 user_id=null 的巨行」。
- **建议**：方案必须补 ① 「日志留存与删除」一节：写清保留 90 天的**执行者是谁**（journald 配置项 / logrotate / 脚本），以及「按 user_id 删除」的手工操作步骤；② 明确匿名口径（`user_id=null` 是否入统计、是否与真实用户混算）；③ 评估是否改用**假名**（HMAC(salt, user_id)，salt 在部署配置里）代替裸 `user_id` —— 分析仍然做得了，但日志泄漏面小得多，这也是与 `0003_auth_media.sql:23-25`「只存摘要」同一条思路。

### 【P1-1】方案要在「各 API route 出口处」加日志，等于把可观测性从一处变成 N 处，且会与现有 request_id 关联断链

- **哪部分有问题**：`proposal.md:8-9`（「在业务层（route handler 里）补充」）。
- **后果**：现在每请求恰好一行、且 `request_id` 由 `server.mjs:57-67` 注入到入站头、业务层复用（`server.mjs:63-67` 注释明确写了「不注入的话业务层会自己再生成一个 —— 用户报编号时根本 grep 不到那一行（实测踩过）」）。如果改在 route handler 出口各写一条，会**变成每个请求两行**（server.mjs 一行 + route 一行），字段口径不同、行序也不保证，`grep request_id` 从「取那一条」变成「取两条并比对」。同时 route handler **拿不到 `client_ip` 与最终 `status`/`duration_ms`**（那些只有 `server.mjs` 的 `res.on('finish')` 里有，`server.mjs:69-103`），所以业务层这条日志会**缺少解析路径最关键的入口信息**。运维侧这是纯粹的倒退：排查时要在两套日志之间做 join。
- **建议**：仍然只在 `server.mjs` 的 `res.on('finish')` 一处出口写，`user_id` 通过**请求作用域传递**（例如 route 里把 `userId` 挂到 res 局部或 AsyncLocalStorage，`finish` 时读取），保持「一请求一行、全字段齐全」。方案 §1 的伪代码位置应当改掉。

### 【P1-2】没有开关、没有降级路径 —— 出问题只能改代码重新发布

- **哪部分有问题**：`proposal.md` 全文未提开关。
- **后果**：日志字段扩张一旦引起写放大（P0-2）或隐私事件（P0-3/P0-4），运维**唯一的手段是改代码、重建镜像、滚动重启**。而 web 是单副本（`docker/compose.yaml:44-76`，`stop_grace_period: 15s`），重启即短时中断。仓库已有现成的开关惯例可循（`context.ts:36-40` 的 `booleanSetting(getDb(), 'forum.enabled', true)`，关掉即对外 404）。
- **建议**：加一个配置项（环境变量或 settings 表）如 `access_log.identity=false`，缺省关闭或可一键关闭「user_id/UA/referer」三字段而**不影响基础访问日志**。这条和 P0-4 的删除流程一起，构成出事时的止血手段。

### 【P1-3】`referer`/`user_agent` 与 nginx 已有日志重复，两条来源会互相打架

- **哪部分有问题**：`proposal.md:22-23`。
- **后果**：`docker/nginx/nginx.conf:20-25` 的 `log_format main` **已经记录** `$http_referer` 与 `$http_user_agent`，且带 `req_id=$http_x_request_id` 可与应用日志 join。方案在应用层再采一遍，形成两个来源、两套字段名、两种保留策略（nginx 日志 vs 应用日志），排查时会出现「nginx 有这条、应用日志没有」或反之的情况。运维要维护两处轮转。
- **建议**：明确「谁是唯一来源」。若只做用户行为分析，建议**只加 `user_id` 一个字段**，其余继续用 nginx 日志按 `req_id` join —— 改动面最小、盘增长最少、也最符合 P0-1/P0-2。

### 【P2-1】与备份/恢复流程完全没有交集，恢复演练时会发现日志不一致

- **哪部分有问题**：`proposal.md:28-30`（存储与保留）。
- **后果**：备份只覆盖 data 与 backups 两个目录（`docker/compose.yaml:66-70`，`BACKUP_ROOT: /app/backups`、`BACKUP_KEEP: 14`）。日志既不在备份内、也不在恢复范围。若将来有人拿日志做业务结论（方案的全部价值就在此），**恢复一次库之后日志与库对不上**，分析结果会不可解释。
- **建议**：在方案里一句话定死「日志不进备份、不参与恢复；分析结论仅对当时日志窗口有效」，并写进 `docs/备份与恢复.md`，避免后续误用。

### 【P2-2】缺少「日志自身」的可观测性

- **哪部分有问题**：`proposal.md` 全文。
- **后果**：现在没有人会知道「日志停止写入」——因为日志不写日志。而方案把产品决策建立在日志完整性上（§2 行为路径还原）。若某次发布后 `user_id` 恒为 null，**没有告警，也没有字段级自检**，等发现时 90 天窗口的数据已经废了。
- **建议**：在 `/api/health` 的自检里加一项「日志最近写入时间」（对齐 `health.ts` 现有 `selfCheck` 口径），超阈值判 warn；并在发布验收清单里加一条「抽样确认 user_id 非空率」。

### 【P3-1】方案文本与仓库约定不一致（影响评审可验证性）

- `proposal.md:14` 写 `ctx.userId`「来自 currentContext()」，实际入口是 `src/app/api/_lib/context.ts:20-25` 的 `currentContext()`，且匿名时为 `anonymousContext`（无 userId）—— 伪代码未处理匿名分支。
- `proposal.md:12-23` 的日志调用是 `log.info('http_request', {...})`，**仓库里不存在这个 `log` 对象**，现有唯一写入器是 `server.mjs:43-47` 的 `writeLine(level, fields)`。按这份伪代码实现，`message` 字段会被覆盖成 `'http_request'` 两次，属于口径冲突。
- 未指向任何需求编号；未说明是否要新增迁移（本方案**不需要**迁移，这点是好的，应在文中明确写出以降低发布风险）。

---

## 三、结论

方向（做行为分析）本身不是运维反对的事，反对的是**这份方案把「日志」当成一个已经存在的、免费的、无上限的资源**——而实际读到的是：日志进 journald 或 30MB 封顶的 json-file（`deploy/dsh-review-web.service:27-28`、`docker/compose.yaml:72-75`），站点磁盘余量闸只有 200MB（`health.ts:50`），审计表刚因为「可被删」被打过补丁（`0042_audit_immutable.sql:19-27`）——**这个仓库的风格恰恰是每一步都写清边界**，这份方案在落点、容量、删除三项上一个边界都没给。

**最低可运维形态**（SRE 视角）：
1. 只加 `user_id`（或以 HMAC 假名替代）**一个**字段，仍只在 `server.mjs` 一处出口写，保持一请求一行；
2. 明确日志落点与保留期的**执行者与配置项**，给出 90 天容量预算数字，超限行为写死；
3. `referer` 去 query、UA 截断，均在写入侧完成；
4. 加可一键关闭新增字段的开关；
5. 补「按 user_id 删除」与「日志不进备份」两句运维口径。

在这五项补齐之前，**不建议按 `proposal.md:8-23` 的伪代码直接实现**。