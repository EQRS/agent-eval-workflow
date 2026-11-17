# 评审：收藏夹条目排序并发更新 · SRE（运维）视角

> 评审对象：`atb/tasks/n9-concurrency/environment/proposal.md`（34 行）
> 评审人身份：**SRE（运维）**（只看发布/回滚/迁移、容量与锁、故障域、可观测性、备份恢复、runbook；
> 不评排序算法、不评前端交互、不重复安全侧的越权结论）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**方案只写了「表结构 + 一段重试循环 + 预期」，一个可运维的系统所需的六件事（迁移、锁、超时预算、限流、指标、回滚）一件都没写；
> 按现文实施，第一次发布就会把一个"无锁竞争"的读路径升级成"写风暴"，且没有任何信号告诉运维它正在发生。**

---

## 摘要（前 5 条最重要）

1. **【严重】§1 的迁移没有编号、没有回填、没有可重入性 —— 这是一次「不可回滚的结构变更」，而 §3 的发布计划是「直接发布」。**
   现场：`proposal.md:10-13`（两条裸 `ALTER TABLE`，无版本号/无事务边界/无回填）、`proposal.md:28-29,31-34`（重试即发布，全文无灰度、无回滚、无观察窗）。
   依据：本仓迁移是**带序号的不可变文件**，执行器按文件名顺序、每个文件单事务执行并登记（`src/server/platform/migrate.ts:93-121`，登记 `schema_migrations` 见 `:95`），
   且 migrate 是迁移的**唯一维护进程**（`docker/compose.yaml:26-33,42`，web/worker 都 `depends_on: service_completed_successfully`，`:46-48,81-83`）。
   现有最高编号是 **0042**（`migrations/0042_audit_immutable.sql`），本方案不占号 → 落地时要么撞号、要么被随便起个名绕过顺序；而迁移**一旦登记就不会重跑**，写错只能再写一个迁移去修。
   后果：① 上线后才发现 position 语义有问题，**只能前滚、不能回滚**；② 回滚代码版本也回不去 schema（`position/version` 列留着，旧代码不认识它们，本身无害，但**新代码写过的数据旧代码读不回来**）；
   ③ 方案要的"用脚本回填 position"完全没写在哪一层跑 —— 若写成"手工跑脚本"，那它就是**一类没有任何登记、没有幂等、不知道跑到哪的运维动作**（见 M-02）。
   建议：编号 0043，`ALTER` 与**回填放同一个迁移文件**（单事务、可重复执行：`ADD COLUMN` 已在 0042 之后、回填用 `UPDATE ... WHERE position = 0` 的幂等形式）；
   回填顺序按既有索引 `created_at DESC, plugin_id`（`0012_interactions.sql:40`、`0019_favorite_folders.sql:29-30`）；
   本地先用 `restore-backup.mjs` 拿一份生产副本演练一遍（见 M-05）。

2. **【严重】§3 的「重试直到成功」在部署层面**没有终止保证**：它把超时预算叠成了「无限 × 5 秒」，而运维没有任何一个开关能让它停。**
   现场：`proposal.md:17-25`（`while (true)`）、`:28-29`（「并发冲突时循环重试，直到成功」）。
   依据：`src/server/platform/db.ts:56` `busy_timeout = 5000`（*单次*遇到写锁最多阻塞 5s）；`db.ts:6-8` 明写 better-sqlite3 是同步驱动、"等待会阻塞所属线程"；
   入口超时 60s（`docker/nginx/snippets/proxy-common.conf:41`）；web 容器 `stop_grace_period: 15s`（`docker/compose.yaml:69`）但 Node 的 I/O 让出之后**同步循环不会被打断**。
   后果：① 一个请求可以吃满 60s 连接（nginx 先 504，用户在页面上看到的是"排序没生效"，服务端还在转）；
   ② 进程被 `docker stop` 时，这个同步循环**没有取消点**，SIGTERM 只能干等；`restart: unless-stopped`（`compose.yaml:68`）下次拉起还是同一个炸弹；
   ③ 运维手上没有"熔断这个功能"的手段（没有 feature flag、没有 maxAttempts 环境变量）。
   建议：删除循环，用**一次** `withTransaction` 整批更新；若保留乐观锁，必须有 `MAX_RETRIES=3`、`ABORT_AFTER_MS` 与 `FAVORITE_ORDER_ENABLED` 开关一起进 `.env`，默认写成"冲突立即 409"（客户端重试成本最低）。
   验收口径要写进发布计划：`p99 < 200ms`、`409 率 < 0.5%`、`无 5xx 增量`。

3. **【高】方案要解决问题的那条读路径，在没有`ORDER BY ... , plugin_id`兜底的情况下，**索引**决定了它是否升级为全表扫 —— 而方案把这件事交给"隐式行序"。**
   现场：`proposal.md:11`（`position ... DEFAULT 0`，无回填、无并列兜底）、`:32`（「顺序不会错乱」）。
   依据：现有读路径依赖 `idx_plugin_favorites_by_user(user_id, created_at DESC, plugin_id)`（`0012:40`）与 `idx_plugin_favorites_by_folder(folder_id, created_at DESC, plugin_id)`（`0019:29-30`）；
   一旦按 `position` 排序，**这两个索引都不再覆盖 ORDER BY**，SQLite 需要把它们当过滤索引 + 一次临时排序；方案里**没有`CREATE INDEX (folder_id, position)`**（对照 `docs/M6素材-收藏夹排序方案.md:16` 那一版是有索引的）。
   后果：`GET /api/me/favorites`（`src/app/api/me/favorites/route.ts:34-53`）会随收藏量线性变慢，且**变慢是渐进的**——压测和上线首日都看不出来，只会在半年后某天被 nginx 504 提醒。
   建议：迁移里同时建 `idx_plugin_favorites_position ON plugin_favorites(user_id, folder_id, position, plugin_id)`（跟随方案实际查询的键）；
   读路径固定写 `ORDER BY position ASC, plugin_id ASC`；把"列表接口 p95"加入 `/api/health` 之外的观测项（本仓 health **刻意不做业务统计**，见 `src/server/platform/health.ts:4-8`，所以别指望它报警）。

4. **【高】全部历史收藏 position 都是 0，**上线那一刻的顺序是未定义的**；而回填脚本是"一次性、无记录、失败无人知"的——这恰好是本仓**最反对**的动作形态。**
   现场：`proposal.md:11`（`DEFAULT 0`）、`:31-34`（预期里没有任何"回填完成"的判据）。
   依据：备份清单 `scripts/backup.mjs:78-88` **不含 `plugin_favorites`** → 这个数据集的"对账基线"在恢复点里根本不存在（见 M-05）；
   仓库对后台任务的既有做法是「有限次数 + 退避 + 每次尝试落 `background_task_attempts`」（`src/worker/handlers.ts:27,284`），单次手工脚本没有这套保护。
   后果：迁移虽然登记了，但**数据层没有到达终态**；如果回填中途失败，运维无法区分"全 0 的行"= 新收藏（合法）还是"没回填到"（缺陷）。
   建议：回填放迁移事务内并在同一迁移里加断言 —— 用一条 `SELECT count(*) ... WHERE position = 0` 之外的判据：
   回填后 `version` 也置 1 并写注释「0 = 未经排序」，这样运维可以在 DB 里用一条 SQL 检查"还剩多少行没被排过"；
   报告这次回填影响的行数（进发布记录），而不是"跑完了"。

5. **【中】方案天然是「一次拖拽 = N 个请求 × N 次重试」，而本仓**所有无状态写接口都挂频控**，唯独这个新接口没有。**
   现场：`proposal.md:4-5,28-29`；全文无频控、无上限。
   依据：`src/server/platform/rate-limit.ts:46-55`（9 个业务动作各一个额度）、典型用法 `enforceRateLimit` + 失败退还（`rate-limit.ts:196-241`）；
   批量接口的既有上限是 200 条（`src/server/modules/interactions/index.ts:1074-1076`）。
   后果：一个用户的一次拖拽就能把 SQLite 唯一写者的队列打满——**故障域不是"这个用户的操作失败"，而是"整站写入变慢"**；而 nginx 侧没有 `limit_req`（`docker/nginx/` 下无任何 `limit_req` 指令），唯一的闸门是应用层频控。
   建议：新增 `favorite_order` 动作（如 30 次/分钟/用户）；配合"整批一次提交"后请求数从 N 降到 1；限流命中数必须出现在日志里（`rate-limit.ts` 已有统一日志出口，别新开一条）。

---

## 完整问题清单（按严重度）

### M-01 【严重】迁移没有编号 / 回填 / 可重入，且发布计划是"直接上线"
- 哪部分：`proposal.md:10-13`、`:28-29`、`:31-34`。
- 后果（文件:行号）：迁移执行器按文件顺序、单文件单事务、登记即不再跑（`src/server/platform/migrate.ts:93-121`）；
  唯一维护进程是 migrate（`docker/compose.yaml:26-33`）；当前最高版本 0042（`migrations/0042_audit_immutable.sql`）。
  方案没占号 → 落地时编号随意；`ADD COLUMN ... NOT NULL DEFAULT 0` 在 SQLite 是元数据操作（快），
  但**方案把"回填"交给了外部脚本**，那一刻事务边界就断了。
- 建议：0043，`ALTER` + 回填 + 索引 + 断言放同一个文件；本地先在生产备份副本上跑一遍再上；
  发布流程里写死顺序：`backup --label pre-0043` → `compose run --rm migrate` → 观察 30 分钟 → 才放开前端。

### M-02 【严重】"用脚本赋 position"是一类无登记、无幂等、无失败可见的运维动作
- 哪部分：`proposal.md:29,32-33`（隐含：回填由脚本完成；方案没写脚本长什么样、在哪跑、失败怎么办）。
- 后果：本仓所有会改数据的长期动作走 `background_tasks`（可重试、有 attempts 表），一次性脚本必须自带幂等与自检；
  否则失败时**schema_migrations 显示"迁移已完成"，数据却停在半路**，而这个矛盾没有任何地方会报出来。
- 建议：把回填写成迁移的一部分（首选）；若确实要独立脚本，则必须：`--dry-run` 先打印行数 → 真跑 → 跑完 `SELECT count(*) WHERE version = 0` 必须为 0 才退出 0 → 结果落 `operation_records`（`migrations/0001_baseline.sql:411-431` 的既有审计载体）。

### M-03 【严重】无限重试没有取消点，直接吃掉优雅停机预算
- 哪部分：`proposal.md:17-25`。
- 后果：`docker/compose.yaml:69` web `stop_grace_period: 15s`；`deploy/dsh-review-web.service:20-21` `Restart=always, RestartSec=3`。
  better-sqlite3 同步循环在进程内不响应信号（`db.ts:6-8`），`docker stop` 到点 SIGKILL → 正在写的事务被硬切，
  WAL 下通常是安全的，但**这次请求的失败没有人知道是谁失败的**（日志只到"UPDATE 0 rows"）。
- 建议：任何重试必须有 `deadline`（如 200ms 预算）；`process.on('SIGTERM')` 后设 `draining` 标志让新请求快速失败；
  这条同样适用于未来任何"服务端自旋"的实现——写进 `docker/daemon.json` 附近的运维说明。

### M-04 【高】超时预算三层不一致，且方案没有对齐任何一层
- 哪部分：`proposal.md:28-29`。
- 后果：nginx `proxy_read_timeout 60s`（`docker/nginx/snippets/proxy-common.conf:41`）> 单次 `busy_timeout 5s`（`db.ts:56`）× 无上限次数；
  最坏情况是"客户端已经 504 了，服务端还在循环"。同时 `src/server/platform/tx.ts:43` 的 `db.transaction()` 是 **DEFERRED**（better-sqlite3 默认），
  读→写升级时同样会撞 `SQLITE_BUSY`，等待同样走 5s；而**全仓 `src/` 没有任何一处处理 `SQLITE_BUSY`**（grep `SQLITE_BUSY` 为 0 命中）→ 它就变成一个 500。
- 建议：写事务统一改 `BEGIN IMMEDIATE`（`tx.ts` 一处改动即覆盖全站，比在每个业务循环里加重试安全得多）；
  应用层显式捕获 `SQLITE_BUSY` → `err.unavailable` 或 `409`，别让它以 500 的形式出现在错误率里。

### M-05 【高】备份/恢复面没覆盖这张表 —— 出事时"回滚到昨天"这件事不成立
- 哪部分：方案全文（无备份/恢复相关表述）。
- 后果：`scripts/backup.mjs:78-88` 的行数清单只有 users/plugins/repositories/reviews/comments/notifications/operation_records/site_settings/configuration_revisions ——
  **没有 `plugin_favorites`**；`src/app/api/admin/backups/route.ts` 走同一脚本（`BACKUP_KEEP` 默认 14，`compose.yaml:62`）。
  库备份本身是全库的（`db.backup()`，`backup.mjs:63-65`），所以**数据不会丢**；丢的是"对账基线"：
  恢复演练时 `countsMatch` 只校验 9 张表（`backup.mjs:114-124`），收藏顺序坏了也全绿。
- 建议：把 `plugin_favorites`（含 `position/version` 两列的存在性）加进 `backup.mjs:78-88` 的清单；
  0043 上线前手动 `node scripts/backup.mjs --label pre-0043`；回滚手册写的是 `restore-backup.mjs --force`（`scripts/restore-backup.mjs:98-111`，会清 `-wal/-shm`），必须同时停 web+worker 再恢复。

### M-06 【高】没有任何指标/日志/告警能回答"这个功能正在坏"
- 哪部分：`proposal.md:31-34`（预期全是"用户感知不到"，没有一条可观测）。
- 后果：`src/server/platform/health.ts:4-8` 明确**不做业务统计**；web 的 HEALTHCHECK 只查 `/api/health`（`docker/web.Dockerfile:76-77`）；
  数据库从 0034 起也没有多套告警通路。于是"排序错乱""重试风暴""锁等待"这类故障**全部只会表现为用户投诉**。
- 建议：至少三件事：① 每个 reorder 请求打一行 structured log（`requestId, userId, folderId, items, attempts, elapsedMs, outcome`）；
  ② 冲突/重试出口计数（可在 `rate_limiter_flexible` 同库落一张日计数表，或直接进现有的访问日志聚合）；
  ③ 给出观察期判据：上线后 24h 内 `attempts > 1` 的比例 < 1%、无 `SQLITE_BUSY`、p99 < 200ms。

### M-07 【中】部署面：这个接口在 `migrate` 之后的新增列上读写，但 web/worker 的**镜像/代码一致性**没有校验手段
- 哪部分：`proposal.md` 全文无部署段。
- 后果：`docker/compose.yaml:32,45` 三个角色共用 `dsh-plugin-review:local` 镜像；`compose up -d --build` 若漏 `--build`，web 跑旧代码、库已是新 schema（`depends_on` 会按新镜像重跑 migrate）→ 旧代码 SELECT * 不认 `position` 能跑，但**新老代码混跑期间顺序语义不一致**，而没有任何地方会报。
- 建议：发布步骤里固定 `docker compose build && docker compose up -d`（或 `--build` 写进 runbook）；`/api/health` 返回的 `schema_version` 与镜像内 `migrations/` 最大文件比对，不一致就 503（`health.ts` 已有 `schema_version` 字段，加一个期望值比较即可）。

### M-08 【中】容量：一个用户的收藏量是**没有上限**的，N 条一次拖拽 = N 个事务
- 哪部分：`proposal.md:4-5`（"批量更新"）、`:17-25`（每请求一行）。
- 后果：仓库里同类批量写接口有 200 条上限（`src/server/modules/interactions/index.ts:1074-1076`），本方案没有；
  而收藏夹条目数本身也无上限（无 LIMIT 约束，见 `FAVORITES_COUNT_FIX.md:410-436` 的同类讨论）。拖 500 条 = 500 个请求。
- 建议：接口层加 `items.length <= 200`（与既有批量接口对齐）+ 服务端单事务；前端做 debounce（拖拽停止 400ms 才发）。

### M-09 【低】时区/时间戳约定没有被方案触及，但 position 的"初始化顺序"依赖 `created_at`
- 哪部分：`proposal.md:11`（position 的语义未定义）。
- 后果：容器与宿主都设 `TZ: Asia/Shanghai`（`docker/compose.yaml:38,53,88`），`created_at` 是整数时间戳；若将来有人把 position 回填写成"按日期分组编号"，时区会让同一天跨零点分裂。
- 建议：position 只表示"夹内相对次序"，**绝不与时间耦合**；文档里写死这条。

### M-10 【低】无 runbook 条目
- 哪部分：方案全文。
- 后果：`deploy/` 下有 backup/discovery-drip 的 unit，但"收藏顺序坏了怎么办"没有对应的一键动作；
  现有 `docs/HANDOFF.md:212-221` 是备份恢复手册，没有业务级回滚条目。
- 建议：在 `docs/` 增一节：症状（用户报顺序乱）→ 判定 SQL（`SELECT folder_id, position, count(*) ... GROUP BY 1,2 HAVING count(*)>1`）→ 处置（重跑回填 / 从 pre-0043 恢复）。

---

## 结论与建议

**这份方案在运维视角下是"半个设计"：它写了表和循环，没写发布、没写锁、没写观测、没写回退，而它要改的恰恰是全站唯一写者 SQLite 的写路径。**

**建议的最小可运维实现（按顺序）**：

1. **迁移 0043**（单文件、单事务、可重入）：`ADD COLUMN position INTEGER NOT NULL DEFAULT 0` + `version INTEGER NOT NULL DEFAULT 0`；
   按 `created_at DESC, plugin_id` 回填并置 `version = 1`（0 = 未排序，便于事后 SQL 对账）；建 `(user_id, folder_id, position, plugin_id)` 索引。
2. **接口**：`POST /api/me/favorites/order`，body `{ folderId, items: [{pluginId, position}] }`，**一次** `withTransaction`（`src/server/platform/tx.ts:42`）内校验归属 + 整批 UPDATE；无循环、无 version 比较。
3. **闸门**：新频控动作 `favorite_order`（30/分钟/用户，`rate-limit.ts:46-55` 的既有出口）；`items.length <= 200`；`FAVORITE_ORDER_ENABLED` 开关进 `.env`，出问题一键关。
4. **发布**：`backup --label pre-0043` → `docker compose build && up -d` → 观察 24h（判据见 M-06）→ 才放开开关。
5. **回退**：关开关即止血（读路径回落到 `created_at DESC, plugin_id`）；数据级回退走 `restore-backup.mjs --force`（先停 web+worker）。
6. **观测**：每请求一行 structured log + 冲突计数；`backup.mjs` 清单补 `plugin_favorites`。

> 评审人声明：本报告只覆盖 SRE/运维视角（发布、迁移、锁与超时、频控、可观测、备份恢复、runbook）。
> 越权与数据完整性由安全视角负责；排序算法与交互由性能/前端视角负责。
> 与本视角强相关且**不论排序怎么设计都必须解决**的是 M-01（迁移形态）、M-03（停机预算）、M-05（备份对账清单）。

> 前提核实：`plugin_favorites` 真实存在且主键为 `(user_id, plugin_id)`（`migrations/0012_interactions.sql:32-37`），`folder_id` 与按夹排序索引在 `0019_favorite_folders.sql:26-30`；
> 方案新增的 `position / version` 两列是**声明的 ALTER**（合理）；但方案声称要修的"并发顺序错乱"在本仓库**没有现场证据**——
> 现有收藏写路径全走 `withTransaction`（`src/server/modules/interactions/index.ts:1081-1121` 等），真正的不一致来源更可能是"逐条请求导致部分成功"。
