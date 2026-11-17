# SRE（运维）评审 · n49-notification-dedup「通知去重与历史清理」

评审对象：`tasks/n49-notification-dedup/environment/proposal.md`（共 31 行）
只从运维角度评：迁移/部署面、调度与执行面、备份恢复与可逆性、容量与可观测性。功能语义与并发正确性由安全/性能两个视角负责，本文只在它影响运维后果时提及。

## 摘要（前 5 条最重要）

1. 【严重】清理任务没有落地通道，而且一旦接了就有"故障任务"的隐患 — 依据：方案 §3 只写"挂在 worker 每日任务里"，但 `background_tasks.task_type` 是 **CHECK 闭集**，只有 `review_check/review_publish/comment_check/comment_publish/plugin_discovery` 五个（`migrations/0007_task_type_plugin_discovery.sql:26`、`src/packages/common/field-rules.ts:88-95`）。塞进队列必然违反约束；而 `claim` 领到它就等于每轮抛一次约束错 — 建议：**不要新建 task_type**。直接复用 `plugin_discovery` 那套"worker 自排程"的模式（`src/worker/scheduled.ts`），把清理做成 `runOnce()` 里的**同步小事务**（例如 `notificationsSweep`），靠 `scheduledTasks` 开关与 `DISCOVERY_SCHEDULE_CHECK_MS` 同频；确实要入队就必须先写 `0007` 式的重建迁移。

2. 【严重】`DELETE FROM notifications` 无视外键删除策略，会把不可删的审计证据一起带走 — 依据：`notifications` 对 `comments`/`content_entities`/`users` 全是 `ON DELETE RESTRICT`（`migrations/0041_role_notifications.sql:38-42`），但**没有任何表反过来引用 notifications**，所以裸 `DELETE` 在 FK 层面是被允许的——它删的是"唯一能证明某个事件发生过"的记录。而同一库里 `operation_records` 被触发器明令禁删（`migrations/0042_audit_immutable.sql`），两套"历史"态度打架 — 建议：清理前先在 `operation_records` 留一条 `action_type='notification.purge'` 的审计（写明删除区间、行数、判据），否则"用户说我 90 天没收到通知"永远无法举证；同时明确写入迁移注释"notifications 是有意可删的、审计以 operation_records 为准"。

3. 【严重】90 天按 `created_at` 一刀切，会删掉未读通知，且这个动作**不可逆** — 依据：`read_at IS NULL` 即未读（`migrations/0041_role_notifications.sql:47`），方案 §2 的判据里完全没有 `read_at` — 建议：判据改为 `created_at < cutoff AND read_at IS NOT NULL`，或者对未读单独放宽（如未读保 365 天）。运维角度更关键的是**先准备回滚**：每日备份只保留 14 份（`deploy/dsh-review-backup.service:26` 的 `--keep 14`），删掉的数据 14 天后连备份里都没有了，所以这条清理上线前必须做到"先 dry-run 报行数、再灰度、再永久"。

4. 【高】清理是长事务式大 DELETE，会顶穿 WAL 并锁住 web 的写路径 — 依据：连接是 `journal_mode=WAL` + `synchronous=FULL` + `busy_timeout=5000`（`src/server/platform/db.ts:52-56`），worker 与 web **共用同一个库文件**（`docker/compose.yaml:65,92`）。一次性删几十万行会让 WAL 文件暴涨、checkpoint 卡住，web 侧写请求可能 `SQLITE_BUSY` 超时；`better-sqlite3` 是同步驱动，主线程会被这条 DELETE 整个占住（`db.ts:7-9` 自己写明了这条纪律）— 建议：**分批删**（每批 `LIMIT 5000`，批间 yield），单批包一个短事务；并给清理单独设一个低峰窗口（参考备份 timer 的 `RandomizedDelaySec=5m` 思路，别和 03:00 备份撞）。

5. 【高】"重复推送"的根因在方案里没有被取证，直接上双轨去重键有引入线上错误的风险 — 依据：站内通知**唯一的投递点**是评论发布事务里的 `deliverCommentNotification`（`src/server/workflows/publish-comment.ts:140`），已经有 `deduplication_key` 唯一键 + `ON CONFLICT DO NOTHING`（`src/server/modules/notifications/index.ts:118-138`）— 建议：上线任何清理前，先在真实库跑一次取证 SQL（按 `recipient_id, kind, content_entity_id` 分组数 > 1 的行），把"到底是不是重复"钉死再动表。运维最怕的是"方案改了写入路径、但问题根本不是这个"。

---

## 详细问题清单

### A. 部署与迁移面

**A1.【严重】方案 §2 的 DELETE 跑在什么 schema 上，方案没说；而 `notifications` 这张表已经被重建过 3 次**
`0008`（建表）→ `0013`（加 detail、放开 comment_id）→ `0016`（放开 content_entity_id）→ `0041`（加 link_url、扩到 10 个 kind）。每次都是 `dsh:migration-transaction=off` + 手工管事务（见 `0016:1`、`0041:1`）。这意味着：
- `notifications` 是本仓**最容易发生"重建表"的表**。任何基于"当前是 0008 结构"写的方案，在下一个 PR 合并后都会失效（`0041:24-26` 已经把"合并顺序是硬约束"血淋淋写在注释里）。
- 方案若真要动 `kind` 闭集，就得再写一次重建迁移，而**重建期间的 FK OFF 窗口是有风险的**（`0007:13-14` 明说"万一中途失败，会留下半成品，执行器的其它迁移有自动回滚，这条没有"）。而这个窗口恰好和 03:00 备份 timer 可能重叠。
- 建议：方案明确"不改 schema，只加一个后台清理"；如果必须改，迁移文件名排在 0043 之后，并在 `.env.example` / 运维手册里补一行"升级前手动跑一次 `scripts/backup.mjs --label pre-deploy`"。

**A2.【高】方案新增的列名与既有列名不一致，直接 INSERT 会因 NOT NULL 约束失败**
方案 §1 写的是 `INSERT ... (id, recipient_id, kind, actor_id, content_entity_id, deduplication_key, created_at)`，但真实表里 `comment_id`、`detail` 的存在是**被 CHECK 锁死的**：`CHECK ((kind IN ('comment_on_content','reply_to_comment')) = (comment_id IS NOT NULL))`（`0041:51`）、`CHECK (kind IN (...8 类...) = (detail IS NOT NULL))`（`0041:53`）。方案这条 INSERT 对评论类通知**必然违反第一组 CHECK**（`comment_id` 为 NULL）。
从运维角度这不只是一处编译错误：它说明方案没有在真实 schema 上验证过，**任何按此写的清理/回填脚本都不能直接上生产**。建议：所有 DDL/DML 上线前先跑 `scripts/verify-migrations.mjs`，并在影子库上跑一次 `acceptance-notifications.mjs`。

**A3.【中】方案 §1 若用 `INSERT OR IGNORE`，会**吞掉** `CHECK (recipient_id <> actor_id)` 的失败**
`INSERT OR IGNORE` 对约束冲突是静默跳过。现有代码刻意用 `ON CONFLICT (deduplication_key) DO NOTHING`（`index.ts:125`）——**只对那一个唯一键容忍**，其它约束照炸。运维视角：`INSERT OR IGNORE` 会把"自己通知自己"这种 bug 变成静默无通知，线上表现为"有人收不到通知"却没有任何错误日志，排查成本极高。建议：沿用 `ON CONFLICT (deduplication_key) DO NOTHING`，并且 `changes===0` 时**必须打日志/计数**，不要静默。

### B. 调度与执行面

**B1.【严重】"挂在 worker 每日任务里"在部署上有两套现实，方案只认了一套**
仓库里有两套部署：
- Docker：`worker` 服务常驻，健康检查看心跳（`docker/compose.yaml:78-108`）；
- systemd：`deploy/dsh-review-worker.service`，`Restart=always`。

而现有的"每日任务"实现方式是 `ensureDailyDiscoveryTask`（`src/worker/scheduled.ts:70-129`）——它**不依赖宿主 cron**，靠 `UNIQUE(task_type, deduplication_key)` + 每轮启动补排实现幂等。方案说"挂在每日任务里"，但没有说明接在哪条通路上。**如果接错**（比如写进某个只跑一次的初始化），清理就永远不会执行，而"历史数据可控"这个目标会静默落空。
建议：显式写成"接在 `src/worker/scheduled.ts` 的 `scheduledTasks` 分支，与 DISCOVERY 同频检查（`DISCOVERY_SCHEDULE_CHECK_MS=60_000`）"，并沿用 `discoveryScheduleSkipReason` 那种"返回原因而不是布尔"的写法，让 `journalctl` 里能直接看到"今天为什么没清理"。

**B2.【高】清理没有开关，出事时运维只能改代码**
现有任务全都留了硬刹车：`DSH_REVIEW_CATALOG_OFFLINE=1`、`DSH_REVIEW_DISCOVERY_DAILY=0`（`scheduled.ts:46-52`），备份也有 `--keep` 参数可调。方案 §2 §3 没有任何"关掉它"的方式。
建议：加 `DSH_REVIEW_NOTIFICATION_PURGE_DAYS`（0 或负数 = 关闭）与 `DSH_REVIEW_NOTIFICATION_PURGE_DRY_RUN=1`，并写进 `docker/.env.example`。这是 SRE 的硬要求：**一个会删数据的定时任务，必须有环境变量刹车**。

**B3.【中】多 worker 进程时的重复执行**
`deploy/dsh-review-worker.service` 是 `Type=simple`，理论上可以 `systemctl start` 起第二个实例；compose 的 `worker` 也可以 `--scale`。任务队列本身靠租约防重（`LEASE_DURATION_MS=120s`，`constants.ts:19`），但方案里的清理如果做成"进程内定时器"而不是"任务入队"，多实例会并发跑同一个 DELETE。
建议：要么走队列（靠租约天然防重），要么在 SQL 层加一把"今天已经清过"的幂等标记（例如写 `site_settings` 一行 `notification.purge.last_date`）。**不要**依赖"线上只跑一个 worker"这个假设。

**B4.【中】删得太慢会顶穿 worker 的优雅退出窗口**
`stop_grace_period: 30s`（`compose.yaml:94`），`TASK_DEADLINE_MS=300s`（`constants.ts:45`）。如果清理做成一条任务，分批删到一半被 SIGTERM，会留下"删了一半"的中间态——虽然对清理幂等，但会留下 `notification.purge` 审计记录不完整。建议：按批提交、每批独立事务，并在审计里用"批次区间"表达，而不是"一次删了 N 行"。

### C. 备份、恢复与可逆性

**C1.【严重】这是一条**不可逆**的删除，而备份窗口只有 14 天**
`deploy/dsh-review-backup.service:26` 的 `--keep 14`，加上 `Persistent=true` 的 timer（`dsh-review-backup.timer:14`）。也就是说：**用户 90 天后发现通知没了，备份里也已经没有 90 天前的数据**。清理一旦跑偏（例如把 cutoff 算错、或时区搞错导致删多了），唯一回滚手段就是"恢复最近 14 天内、且还没被覆盖的备份"，代价是整个库回退。
建议：(a) 上线第一次执行前手工跑 `node scripts/backup.mjs --label pre-purge --keep 30` 留一份**单独的、不被日备轮转覆盖**的恢复点；(b) 判据里必须用 UTC 计算并在日志里打印 cutoff 的具体时间戳，别用"90*86400_000"这种无法复核的表达式。

**C2.【高】备份的行数清单里已经包含 `notifications`，但清理后行数会"看起来正常地缩水"**
`scripts/backup.mjs:78-94` 的 `counts` 里明确列了 `notifications`，`manifest.json` 记它、自检也对它（`countsMatch`）。清理上线后，运维在 `journalctl -u dsh-review-backup` 里会看到 `notifications` 从几十万掉到几千——**这是正常行为，但如果清理逻辑出错或被人误跑，外观完全一样**。
建议：让清理把"每次删除行数"写进 `operation_records` 与结构化日志（`src/server/platform/log.ts`），备份 manifest 才能和它对照；否则运维没法区分"计划内清理"与"数据被误删"。

**C3.【中】恢复演练台账要同步更新**
`src/app/api/admin/backups/route.ts` 与 `admin-backup-panel.tsx` 背后是一条"恢复只能在维护窗口手工执行"的流程。加了清理之后，恢复流程多了一个前提："恢复到的这份备份，里面通知还全不全"。建议在运维手册里补一句：**执行恢复时，通知数据以备份时点为准，晚于备份时点的通知不可恢复**。

### D. 容量与可观测性

**D1.【高】"历史数据越来越多"是现象，不是根因——清理之前应该先量**
方案 §背景直接给了结论（"越来越多"），但没有任何量级：现在 `notifications` 多少行？`kind` 分布？`read_at` 未读占比？360 天以上占多少？——从运维角度，**90 天这个数字没有任何依据**。如果 95% 的通知都在 90 天内被读过，那清理掉的根本不多，"数据可控"只是心理安慰；反之如果表已经几百 MB，那该做的是建索引/归档表，而不是 DELETE。
建议：方案附一条取证 SQL 与实测结果。现有索引只有 `idx_notifications_inbox ON notifications(recipient_id, read_at, created_at DESC, id)`（`0041:69`）——**没有单独 `created_at` 的索引**，所以 `WHERE created_at < ?` 是**全表扫**。删之前先扫一遍全表，这个成本方案里完全没提。建议清理判据复用它（`recipient_id` 不可知，所以直接加一条 `CREATE INDEX idx_notifications_created ON notifications(created_at)`，或者在清理时接受一次全表扫并放在低峰）。

**D2.【中】清理没有可观测性设计**
没有任何指标能回答"昨天清理跑了吗、删了多少、耗时多久"。现有 worker 的心跳文件（`src/server/platform/heartbeat.ts`，被 `scripts/worker-healthcheck.mjs` 读）只看"进程活着"，不看"定时任务跑过"。
建议：清理完成后写一条结构化日志（`{task:'notification_purge', cutoff, deleted, batches, duration_ms}`），并在后台页（类似 `admin-discovery-panel.tsx`）给一个"最近一次清理"的展示面板。

**D3.【低】WAL 空间与磁盘水位**
一次性大 DELETE 会让 `app.sqlite-wal` 显著增长，而 compose 里没有任何磁盘水位告警。建议：清理后主动做一次 `PRAGMA wal_checkpoint(TRUNCATE)`，或者接受"数据目录会在清理窗口内短暂变大"并在运维手册里写明。

### E. 与既有约束的冲突（运维视角会挨的第一刀）

**E1.【中】`kind` 仍是 CHECK 闭集（10 项），方案如果想加"去重命中"之类的新 kind，必须重建表**
`0041:39`。且这次重建会撞上 `0041:24-26` 已经写死的"合并顺序是硬约束"那个雷。建议：**不要新增 kind**。

**E2.【中】`defer_foreign_keys = ON` 是连接级设置**
`src/server/platform/db.ts:50`。清理如果在事务里批量删除，外键延迟校验到 COMMIT 才发生——如果哪天有人给 `notifications` 加了反向引用（比如"通知已读回执"表），这个 DELETE 会在 COMMIT 时炸而不是在语句处炸，排查会很难。建议：清理脚本显式不依赖 defer，或在批处理事务里先 `PRAGMA defer_foreign_keys = OFF`。

---

## 附：给方案的 SRE 侧硬性要求（可操作的 6 条）

| # | 要求 | 达不成时的后果 |
|---|---|---|
| 1 | 不加新 `task_type`，接 `scheduled.ts` 的同步清扫通路 | 队列满屏约束错，claim 失败 |
| 2 | 清理前必须留一条 `operation_records`（`notification.purge`）审计 | 无法举证"谁在什么时候删了什么" |
| 3 | 必须有 `DSH_REVIEW_NOTIFICATION_PURGE_DAYS` / `..._DRY_RUN` 两个 env 刹车 | 出事只能回滚代码发版 |
| 4 | 分批删（`LIMIT` + 批间 yield），单批短事务，低峰窗口 | WAL 暴涨 + web 写请求 `SQLITE_BUSY` |
| 5 | 保留未读（`read_at IS NOT NULL` 才删）；删除量写进日志 | 90 天未登录的用户回来一片空白 |
| 6 | 首次上线前手工留一份不被轮转覆盖的 pre-purge 备份 | 删错了无法恢复（日备只留 14 天） |

---

## 判据覆盖（供评分参考）

criteria.md 的 L3/L4 条目里，SRE 视角直接覆盖的有：
- "清理 90 天会删除未读通知" → 本文 **3**（并补了"备份只留 14 天所以不可逆"这一层）
- "清理删通知不删关联数据，但审计会要求可追溯" → 本文 **2 / C2 / 附-2**
- "真实的去重键已经是 deduplication_key 且有 UNIQUE" → 本文 **5**（从运维后果切入：先取证再动表）
- "kind 是 CHECK 闭集" → 本文 **E1**（运维后果：再建一次表、FK OFF 窗口风险）
- "有 CHECK (recipient_id <> actor_id)" → 本文 **A3**（`INSERT OR IGNORE` 会静默吞掉它）

超纲补充（经核实成立）：**B1** 双部署通路、**B2** 无刹车、**B3** 多实例重复执行、**B4/C3** 优雅退出与恢复演练台账、**D1** 无 `created_at` 索引导致全表扫、**A2** 方案 INSERT 列与真实 schema 不符。
