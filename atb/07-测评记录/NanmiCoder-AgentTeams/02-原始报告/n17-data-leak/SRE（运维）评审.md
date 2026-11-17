# n17-data-leak SRE（运维）评审

> t24 · e3-investigator · 2026-11-17

---

# n17-data-leak SRE（运维）评审

评审对象：atb/tasks/n17-data-leak/environment/proposal.md（38 行）
已对仓库逐行核实（server.mjs、deploy/*.service、docker/compose.yaml、docker/daemon.json、
src/server/platform/log.ts、src/server/platform/request-id.ts、src/server/platform/client-ip.mjs）。

## 最重要 5 条

### S1【阻塞】30 天都存不下，更别说 90 天 —— 现有日志管道是**有损**的
proposal:30「日志写入 JSONL 文件，保留 90 天」。
**代码事实**：本仓有两套部署，**两套都会把日志自动丢掉，且丢的过程不可见**：
  - Docker（docker/compose.yaml:71-74 / 96-99 / 128-131 与 docker/daemon.json:3-7）：
    `driver: json-file, max-size: 10m, max-file: 3` → **每个容器硬上限 30MB**，超出即轮转并删除最旧；
  - systemd（deploy/dsh-review-web.service:29-30、dsh-review-worker.service:23-24）：
    `StandardOutput=journal` → 进 journald，而 deploy/ 下**没有任何 journald 配置**
    （grep 无 `Storage=`），多数发行版默认 `Storage=auto` 且受 `SystemMaxUse` 限制，容器/机器重启即可能清空。
② 后果：proposal:36「能看到用户完整的浏览路径」与 :37「保留 90 天便于回溯」**都不可能成立**。
更糟的是失败是**静默**的：分析脚本跑出来的是“部分用户的残缺路径”，
会得出“用户到 X 页就流失了”这类**错误结论**，而没有任何报错提示数据被截断。
③ 建议：不要让分析数据依赖 stdout。两条路选一：
  (a) 写**独立文件**到 `$DSH_REVIEW_DATA_DIR`（compose.yaml:41 已挂载 `/app/.data`），
      自带按天切分 + `keep 90` 的清理（与 scripts/backup.mjs:148-153 的 keep 逻辑同构）；
  (b) 落库（SQLite 一张 `access_events` 表 + 按天分区清理），便于 `user_id` 聚合查询。
无论哪条，都必须显式给出**容量估算与上限**（见 S2），并加“写入失败要告警”而不是静默。

### S2【阻塞】方案说“成本可控”但没有算过 —— 90 天的量级与磁盘预算
proposal:38「数据量大但都是文本，成本可控」。
**代码事实**：现有每请求一行已含 request_id/client_ip/method/path/status/duration_ms（server.mjs:67-81）。
方案要**再加三个字段**（proposal:20-22：user_id、user_agent、referer），
而 `user_agent` 与 `referer` 是**最长且最不可控**的两个字段（UA 常见 100-200 字节，referer 可上千）。
② 后果：按每行 +400 字节保守估，1 万请求/天 → 约 4MB/天 → 90 天约 **360MB**；
若流量到 10 万请求/天 → **3.6GB**。这些还只是原始文件，未计索引/分析中间产物。
与 compose 的 30MB 上限相比是 **12 倍到 120 倍**的差距——不是“可控”，是必须改基础设施。
且磁盘写满会让 **SQLite 写入失败**（WAL 也在这块盘上），直接拖垮业务，
这不是“日志问题”而是**可用性事故**。
③ 建议：方案必须写出 (1) 日请求量假设；(2) 每行字节上限（UA/referer 截断，如各 256 字节）；
(3) 90 天总容量；(4) 磁盘水位告警与“写满前自动降级（只留抽样）”的策略。

### S3【高】“在各 API route 出口补记一条”会**重复计数**，且漏掉页面请求
proposal:10-13「在业务层（route handler 里）补记一条」。
**代码事实**：server.mjs:62-82 已经在 `res.on('finish')` 为**每个请求**记一行（含状态码与耗时）。
业务层若再记一条 `http_request`，同一请求就有**两行同名字段**。
同时 server.mjs:35-37 的静音规则只覆盖 `/_next/`、`/api/health`、`/favicon.ico` ——
**页面请求（GET /plugins 等）也会进日志**，而 route handler 只覆盖 `/api/*`。
② 后果：(a) 聚合脚本按 `user_id` 统计时，API 请求被算两次 → 路径频次失真；
(b) 页面访问（占流量大头、也最能反映“浏览路径”）**完全没有 user_id**，
proposal:36「完整的浏览路径」只有 API 那一半。
③ 建议：**不要**在 route handler 里另记一条。正确做法是把 `user_id` 灌进既有那条日志：
如 server.mjs:59 已经用 `req.headers['x-request-id']` 注入做 precedent，可同样由业务层在
一个固定的响应头（如 `x-actor-id`）回传，server.mjs 在 `res.on('finish')` 里读它补进同一行。
这样“一个请求一行”的不变量不破，且页面+API 同一口径。

### S4【高】user_id 不是走“信任边界”进来的，与仓库既有纪律冲突
proposal:20 注释「来自 currentContext()」，但方案没说这条日志**由谁最终写出**。
**代码事实**：本仓对“客户端可写的头不可信”有明确纪律 ——
src/server/platform/client-ip.mjs:7-10：「反代后面如果直接取 x-forwarded-for，
等于让**客户端自己决定**日志里记的是谁（这个头 HTTP 上谁都能写）」，
所以默认只用 TCP 对端地址；compose.yaml:57-58 也为此限制 TRUST_PROXY。
同样地，request_id 的入站值也要过 `isSafeRequestId`（request-id.ts:23,36-38）才允许沿用，
注释 :33-34 理由是「若允许换行或超长内容，会造成日志注入」。
② 后果：若 user_id 经由**响应头/入站头**传递而不校验，客户端可伪造任意 user_id，
把行为分析污染成“某人访问了某页”——这在合规事故调查时是**反证材料**。
③ 建议：若走 S3 的响应头方案，该头必须由服务端在**服务端上下文**写入（不是回显入站值），
且 server.mjs 只接受**已知格式的 ID**（UUIDv7，可复用 request-id.ts:23 的正则思路），
格式不符就丢弃并计一次告警，而不是原样落盘。

### S5【高】referer / user_agent 未截断、未转义 —— 日志注入与 PII 扩散
proposal:21-22 直接取 `req.headers['user-agent']` 与 `referer`。
**代码事实**：现有 writeLine 用 `JSON.stringify`（server.mjs:40、log.ts:13-18），
这对换行注入是**有效的**（引用会转义），这点没问题；但 `log.ts:24` 的脱敏黑名单是
`/token|secret|password|credential|authorization|cookie|code_verifier/i`，**按字段名匹配**，
`referer` 与 `user_agent` 都不会命中。
② 后果：(a) referer 里常带 query string（`?token=`、`?email=`、重置密码链接），
会**原样落进 90 天日志**并可能被分析脚本导出；
(b) 无长度上限 → 单个超长头即可让单行撑到几百 KB（Node 默认 header 上限 16KB，仍可观），
加速 S2 的磁盘耗尽。
③ 建议：referer 只保留 **origin + path**，丢弃 query（与 m13 导出的白名单纪律一致）；
user_agent 截断到 256 字节；两者都做显式长度校验后落盘。

## 完整清单（其余）

### S6【高】90 天保留与仓库“最小化”取向不符，且方案缺删除机制
proposal:30 只写“保留 90 天”，没有写**谁删、怎么删、删失败怎么办**。
**代码事实**：本仓对数据的既有取向是**尽量少留**：备份只留 14 份（compose.yaml:62 `BACKUP_KEEP: '14'`，
scripts/backup.mjs:38,148-153 的 keep 清理）；通知/审计都有明确的生命周期。
② 后果：没有清理任务则日志**永久增长**（S2 的磁盘耗尽最终会发生）；
有清理但没监控则“清理失败”也会静默走到同一个结局。
③ 建议：清理挂到既有每日调度（deploy/dsh-review-backup.timer:12 已是 03:00，可复用同一时点），
并在清理后**打一行汇总日志**（删了几份、剩几天）——否则无法判断清理是否真的在跑。

### S7【中】“按 user_id 还原浏览路径”= 持续的个人追踪，方案未做任何收敛
proposal:27「按 user_id 聚合，还原每个用户的页面访问序列」。
② 后果：这是把**匿名可得的统计数据**升级成**逐人行为档案**，且保留 90 天。
从运维角度它还有额外代价：任何一次日志外带（误发、备份泄漏、临时排障拷走）都是全站用户轨迹。
③ 建议：优先考虑**去掉 user_id** 的方案——用 `session_id`（随会话轮换）做路径还原，
完全够回答“用户从 A 页到 B 页的转化”，且无法长期锁定到具体个人；
若确实需要 user_id，至少做**假名化**（HMAC(user_id, 部署密钥)），
分析脚本可用、日志泄漏时不可反查。

### S8【中】分析脚本读 90 天 JSONL 会是内存/IO 炸弹
proposal:33「用脚本按 user_id 分组，统计路径频次」。
② 后果：360MB-3.6GB（S2）的 JSONL 一次性读进内存分组，在 1-2GB 的便宜 VPS 上会 OOM；
而这种脚本通常跑在**生产同机**上，OOM 会连带影响 web/worker。
③ 建议：脚本做**流式**处理（逐行读 + 增量聚合，`readline` 即可），并加 `--since/--until` 限定窗口；
不要在分析时把全量读进内存。

### S9【中】方案与 server.mjs 的既有静音规则没有对齐
server.mjs:35-37 静音 `/_next/`、`/api/health`、`/favicon.ico`，注释 :13 写明理由是
「量最大、信息量最小，会把日志淹掉」。
② 后果：若分析数据从同一管道来，`/_next/` 的静态请求（占日志量绝大多数）不在其中，
这在行为分析里是对的；但**方案没说清**，实现者可能“顺手”把静音去掉以求“完整”，
结果磁盘先炸（S2）。
③ 建议：明确复用现有静音规则，并在文档写明“分析只覆盖非静态请求”。

### S10【低】日志字段命名与既有 access log 不一致
proposal:14-23 用 `route_path`，而 server.mjs:77 用 `path`；两者同义不同名，
会让 grep/聚合脚本要处理两种键。建议统一为 `path`，或在 server.mjs 那一行扩展（见 S3）从而根本不需要新键。

## 一句结论
“想知道用户怎么用”是合理需求，但本方案把它做成了一条**有损、无上限、含 PII 的日志管道**：
现有两套部署分别只有 30MB（docker）与不定（journald）的日志上限，
**90 天与“完整路径”都做不到**；再叠加 route handler 重复计数、user_id 信任边界缺失、
referer 未脱敏四件事，结果既拿不到可靠数据，又扩大了数据暴露面与磁盘耗尽风险。
建议先按 S1-S5 重做设计（独立文件/落库 + 单行不变量 + 假名化 + 脱敏截断），再谈分析。