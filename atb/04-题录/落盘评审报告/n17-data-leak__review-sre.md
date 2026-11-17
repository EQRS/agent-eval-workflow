## 摘要（前 5 条最重要）

1. 【严重】方案把带 user_id 的访问日志「写 JSONL 文件保留 90 天」，但本仓库**没有为这一类日志提供任何轮转、配额或磁盘监控**：systemd 一套走 journal（deploy/dsh-review-web.service:29-31，无 Storage=/SystemMaxUse= 约束），compose 一套走 json-file（docker/compose.yaml:70-74、docker/daemon.json:1-8）。如果把新增的持久化日志写到 90 天，单是磁盘占满就会拖垮 SQLite 和备份。建议改为写 journald 或 json-file，并在方案里写明配额；或者单独用 logrotate（daily、rotate 90、maxsize 100M、compress）配置，并把日志容量纳入 /api/health 和告警。
2. 【严重】写入位置选错了层：这些日志会混进 **web 容器的 stdout**，而 nginx 对同一请求也按 log_format main 全量记录 referer 与 UA（docker/nginx/nginx.conf:24-29），于是每个请求的 UA、referer 各存两份，还和运维排障用的访问日志共用同一个文件（server.mjs:34-37 的静音规则也只对 server 的日志生效）。日志本该进日志系统，不该由业务进程自己写文件。建议：把方案 §3 改成「扩展既有 http_request 结构、走 stdout」，或者由 nginx 统一接管；不要在容器里再另开文件（容器层重启即丢，宿主上又是 /root 之外留痕）。
3. 【高】方案 §1 让每个 route handler 各补记一条 http_request，会与 server.mjs:62-82 每请求写的那条**重复**（同一 message 名、同一 request_id），还会漏掉没有经过 handler 的请求（例如 /media、404、鉴权失败的请求）。如果按 90 天留存，还会把日志量翻倍、拉长排查时的干扰。建议只在 server 层扩展字段来源（例如让 handler 回写 res.locals.userId，在 finish 时统一打），handler 内不写访问日志。
4. 【高】方案没有定义**字段契约和脱敏路径**：log.ts:24 的敏感键黑名单（token|secret|password|credential|authorization|cookie|code_verifier）只按键名匹配，**user_agent、referer 都不会被它过滤**。UA 和 referer 经常带第三方查询参数（OAuth callback 里的 code、邮件里的 token）；referer 还会把用户在站内的来源页带出去。建议在采集端就只保留白名单内的机器字段（UA 的浏览器/设备族），referer 只留 origin+path，并对全仓的日志输出做一次统一的脱敏回归测试。
5. 【中】第 3 点「数据量大但都是文本，成本可控」这个判断缺依据，实际上不可控：compose 下 json-file 的 10m×3 只够存几分钟到几小时（同结论见 tasks/n5-incident-rca/review-perf.md:45），systemd 下的 journald 默认还有 RateLimitBurst 500 条/30s 的限流，日志高峰时会被整段丢弃。本方案最需要的正是高峰时段的完整序列，而它恰恰存活不下来。建议在方案里补上「日志存储的边界」一节，明确写出容量预算、保留策略和高峰丢弃时的降级取舍。

---

## 完整评审（SRE / 运维视角）

### A. 日志生命周期与容量（本方案最核心的运维风险）

| 项 | 现状证据 | 方案缺口 |
|---|---|---|
| 无轮转、无总量上限 | 方案 §3 只写「JSONL 保留 90 天」，仓库里没有对应的 logrotate、tmpfiles 或 journald 配置 | 90 天之后凭什么删除？靠什么保证不会写满盘？ |
| 两套部署面口径不一致 | systemd 走 journal（deploy/dsh-review-web.service:29-31），compose 走 json-file（docker/compose.yaml:70-74、docker/daemon.json:1-8） | 方案没说明要在哪种拓扑上生效，两种拓扑下 90 天都做不到 |
| 磁盘余量告警阈值很高 | src/server/platform/health.ts:50、88-97：剩余 <200MB 即 fail(503) | 日志一旦把分区写满，健康检查变成 503，编排器会重启 web，**看似在自愈，实际在掩盖问题**，而且重启也清不掉日志 |
| 备份会一起受损 | scripts/backup.mjs:99-107 要打包 media.tar.gz；磁盘满时备份在 integrity/counts 那步失败，退出码非 0（backup.mjs:159、179） | 方案没提「日志增长会连带把恢复点体系打坏」，也没提要不要把日志目录排除在备份之外 |
| 与既有告警共用文件 | server.mjs:34-37 只把 /_next/*、/api/health、/favicon 静音 | 90 天的行为日志会把「今天这次 500 的那一段」挤出去，排障窗口反而**变小** |

建议的运维形态（供方案采纳）：
- 不要在容器内写文件；**stdout 就是日志的原生出路**，采集由外部负责（docker json-file / journald → 采集端）。
- 若必须自己保留更久，则使用**独立卷 + logrotate**（daily、maxsize 100M、rotate 90、compress、copytruncate），并在 /api/health 增加「日志卷剩余空间」的检查项（复用 health.ts 的 disk 检查思路）。
- 保留期与采样率必须是**两个独立的旋钮**：结论「看路径频次」不需要逐行存 90 天，采样或者是预聚合（按 user_id × route_path × 天做计数）就能得到同样的结论，数据量能降 2–3 个数量级。

### B. 写入路径与重复记录

- server.mjs:62-82 已经在每个请求的 finish 事件里写一条 `message:'http_request'`。方案 §1 在 route handler 出口再写一条同名日志，结果是同一 request_id 出现两行，grep 时要靠字段差异区分，排障时极容易误读。
- 在 handler 内串行写日志还会把日志 I/O 放进请求的关键路径（这个方案没提到同步 write 的退路：stdout 是异步的，文件 fs 直写是同步的）。如果后续改成直写文件，在慢磁盘上会直接抬高 p95，而 server.mjs:18 的慢请求阈值只有 1000ms。
- 采纳建议：**把「用户上下文」做成 server 层可读的东西**，在 res 'finish' 时统一补充字段，只在 server.mjs 一处打日志。

### C. 请求链路上的连乘放大

- 部署面 nginx 是唯一入口（docker/nginx/conf.d/default.conf:14-44），并按 log_format main 记录了 referer 与 UA（nginx.conf:24-29）。方案再加一份到应用日志，等于**入口和业务各存一份**，两份的保留策略还不一样，日后没有单一事实来源。
- server.mjs:34-37 的静音规则意味着 /_next/、/favicon 在应用侧是静音的，但在 nginx 侧并非全静音（site-locations.conf:22-27 只有静态块 off）。方案若按「所有 API route 出口补记」来实施，页面访问序列里的静态噪声会重新出现，却又不完整。
- 部署文档里的运维排查步骤是 `journalctl -u dsh-review-web | grep <id>`（docs/项目规划/部署-README.md:203-209），新增 90 天文件日志之后，这个步骤要不要扩展成「两处都 grep」是方案必须回答的问题，否则 runbook 会失效。

### D. 变更与回滚（SRE 关心的是「能不能退回去」）

- 方案没写**开关**：没有环境变量（例如 DSH_REVIEW_AUDIT_LOG=1）就直接改代码上线，出问题时只能回滚构建。
- 也没写**上线顺序**：web 是 Restart=always（deploy/dsh-review-web.service:20-21），如果新增字段引发异常，会进入重启循环；而 server.mjs 每请求都写日志，任何一次序列化异常（例如 user_agent 是数组导致 JSON.stringify 正常，但某些自定义字段是 BigInt 就会抛异常）都会让请求失败。建议方案里补上：字段序列化失败时必须**降级跳过该行**，不能中断请求。
- 数据保留 90 天意味着**上线即生效的数据留存义务**：一旦上线，前 30 天的数据就不可撤销了。建议上线前先确认「回滚意味着这批已落盘的数据怎么处理」——这是方案里最容易被漏掉、但实操中一定有人问的一点。

### E. 与既有契约的冲突

- src/server/platform/log.ts:1-6 的头注释已经立了规则：「不记录凭据、会话 token、**私有正文**或第三方原始响应」。把原始 UA/referer 原样落盘 90 天，属于对这条规则的**扩写**，而不是新增特例。方案没有提到要同步更新这条文档约定，也没有提脱敏白名单设计；同时 log.ts:24 的黑名单也覆盖不到 UA/referer 内嵌的第三方 token。按仓库的「算法权威唯一」思路，日志契约必须先在 log.ts 里写成唯一权威，再在 handler 中引用，不能在 handler 里各自拼字段。
- health.ts:1-8 明确「健康检查不做业务统计，端点可能公网可达」。如果方案后续想「在首页看用户路径」，绝不能顺手把它加进 /api/health。

### F. 安全边界（越界提醒，主审请裁量）

仅列一条与运维直接相关、确定性高的：方案把 user_id 与 IP 一起长期留存（IP 来自 server.mjs:71-75 的 client_ip，已归一化），在 IPv6 下 $remote_addr 可能是**全地址**而非 /64 掩码。如确需按人分析，建议先做 IP 截断（IPv4 /24、IPv6 /64），这样既能让排查仍然可用，也能减少留存面的隐私争议。完整的隐私/合规结论应由安全与法务视角给出。

### G. 可观测性反哺：保留期内的运维查询

方案 §4「用脚本按 user_id 分组统计路径频次」，在 90 天 × 全量的规模下，每次统计都要全表扫描（grep 或 zcat），会与 web 争 IO。建议用**按天分文件 + 预聚合**：每天的 `counts-YYYYMMDD.jsonl`（user_id, route_path 计数）是查询面，原始行只保留 7 天供排障使用。这与 health.ts 的「高频调用不该成为负担」是同一个取舍。

### H. 运维验收判据（建议写进方案 §预期）

1. `df` 显示日志目录在第 90 天仍不触及 200MB 的告警线（或由配额强制保证）；
2. `docker compose logs web | grep http_request` 与 `journalctl -u dsh-review-web` **字段一致**，且同一 request_id 只有一行；
3. 关掉开关后重启，行为完全回退（包括不再产生新文件）；
4. 高峰压测下 journald 不丢行（调大 RateLimitBurst），或者明确接受丢弃量并记录到方案里；
5. 备份耗时与体积不因日志增长而变化（日志不进 backup）。