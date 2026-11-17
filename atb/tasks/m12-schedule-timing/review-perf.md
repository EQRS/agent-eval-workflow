# 评审：定时任务调度时间改造 · 性能工程师视角

> 评审对象：`atb/tasks/m12-schedule-timing/environment/proposal.md`（50 行）
> 评审人身份：**性能工程师**（只看执行时序、配额与 IO 成本、索引实效、索引与队列写放大、容量与可观测；不评权限模型）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**方案会让定时任务从「每天最多一轮」退化成「每天 + 每次 tick 都尝试一轮」**，
> 且在 03:00 与每日备份撞点。它新增的 `scheduled_at` 列与索引**在方案自己的代码里一次都没被使用**。

---

## 摘要（前 5 条最重要）

1. 【严重】**§2 的 `if (now >= target)` 是「过期就补跑」，不是「到点才跑」——重启一次补跑一轮，停机一周补跑七轮。**
   依据：`proposal.md:31-32`；daily key 只用 `today` 计算（`proposal.md:29,35`），所以**当天只会被去重一次**，
   但**跨天不会**：worker 停机 7 天后启动，`now >= target` 对当天成立 → 立刻排入当天这一轮，
   而 `dedup: discovery:github:${d}`（`:24,35`）每天一个键 → 每跨一天就多一轮（如果 tick 循环里按天推进的话）；
   更糟的是**没有容差窗口**时，03:00 的任务在 12:00 启动也会立刻跑，**配额被消耗在业务价值最低的时刻**。
   量化：发现任务单轮实测约 26 次 GitHub 核心请求（`deploy/discovery-drip.sh:21-22`），未认证配额仅 60 次/小时（`src/server/modules/catalog/discovery.ts:49-51`）——
   03:00 与 04:00 两个 schedule 在同一 tick 内触发，`deploy/discovery-drip.sh` 之外还会叠加 warmup 的请求。
   建议：**加触发窗口**（`now - target <= WINDOW_MS` 才排，如 15-30 分钟），并**一次性推进到下一个 target**，
   而不是每个 tick 重新判断；窗口外直接跳过并记一条「错过」日志（与 `deploy/dsh-review-backup.timer:14` 的 `Persistent=true` 语义相反——
   备份要补跑，发现不需要）。

2. 【严重】**§1 的 `scheduled_at` 列 + `idx_tasks_scheduled` 是「死代码」：方案自己的 tick 根本不查表。**
   依据：`proposal.md:15-16` 建列与索引；`proposal.md:28-41` 的 tick **遍历内存数组** `schedules` 算 `target`，
   用的是 `deduplicationKey` 去重，**从不读也不写 `scheduled_at`**。
   后果：多一列（`background_tasks` 已有 21 列、`0007_task_type_plugin_discovery.sql:24-59`）与一个恒不被使用的索引，
   而 `background_tasks` 是全站写热点（每次 claim/renew/finish 都 UPDATE，`src/server/modules/tasks/service.ts:246-379`），
   **每多一个索引，所有写都多一次维护成本**，换来零收益。
   建议：二选一——① 删掉 §1，调度完全在内存里算（当前设计就是这样）；
   ② 若真要让 `scheduled_at` 有意义，则改由 **`available_at` 表达到点时间**（既有列，`0001_baseline.sql:359`，已是 claim 的排序键），
   调度器负责「提前把明天的任务排进队列」而不是「到点才 enqueue」——见摘要 3。

3. 【高】**「到点才 enqueue」把调度延迟绑在最长 `IDLE_SLEEP_MS` 上，且与「提前排队」相比没有收益。**
   依据：`src/worker/main.ts:208-220`（无任务时 `await sleep(IDLE_SLEEP_MS)`）、`src/server/modules/tasks/constants.ts:25,48`（`IDLE_SLEEP_MS = RECOVERY_SCAN_INTERVAL_MS = 10s`）；
   `proposal.md:20`「复用现有主循环的节拍」= 每 10 秒一次 tick。
   后果：到点误差 0-10s（可接受），但**如果 worker 正在跑一个长任务**（比如发现任务本身，单轮可能上百秒，
   `src/server/modules/tasks/constants.ts:45` `TASK_DEADLINE_MS=300s`），主循环卡在 `execute()` 里，
   下一个 tick 要等任务结束——**04:00 的 warmup 可能被 03:00 的发现任务拖到 03:05 之后**。
   建议：调度 tick 与任务执行解耦（tick 放在 `setInterval` 里，不放在 `runOnce()` 的串行路径上）；
   或直接改成「启动时把未来 24h 的任务按 `available_at` 排好」，worker 只需 `claim`，
   既消掉 tick 空转，也消掉「执行中被拖」的问题。

4. 【高】**与既有 systemd timer 的 03:00 备份正面撞车。**
   依据：`proposal.md:24` 的 `daily_discovery` 定在 `03:00`；`deploy/dsh-review-backup.timer:12` `OnCalendar=*-*-* 03:00:00`、
   `:13` `RandomizedDelaySec=5m`、`dsh-review-backup.service:26` 跑 `docker compose run --rm --no-deps web node scripts/backup.mjs --keep 14`。
   后果：备份要**整库拷贝 + integrity check + 媒体清单**（同一份 SQLite 文件，`:22-28`），发现任务要**持续写库 + 长时间持有网络连接**；
   两者在同一分钟起跑，备份的 `integrity_check` 读到的是一个正在高频写的库；
   而 SQLite 是单写者（`src/server/platform/db.ts` 开 WAL），备份期间的写入会被 checkpoint 拖慢。
   建议：把发现任务挪到与备份不重叠的时段（如 05:00），或像 `dsh-review-backup.timer:13` 那样给自己加随机偏移；
   并在方案里写明「为什么选这个时刻」（现有备份文件头 `:1-7` 就是这么写的，可照抄格式）。

5. 【中】§3 用一句话带过「新增 task_type」，实际要重建 `background_tasks` 全表——**这是一次全量 IO + 停写窗口**，不是一个常量。
   依据：`proposal.md:44-45`；`migrations/0007_task_type_plugin_discovery.sql:1-19`（该文件头已经写明：必须 `PRAGMA foreign_keys=OFF`、必须 `dsh:migration-transaction=off`、
   中途失败会留半成品）、`:62-77`（全表 `INSERT ... SELECT` + `DROP` + `RENAME`）、`:79-81`（索引随表被删，必须重建）。
   更关键的是：`background_tasks` 被 `background_task_attempts.task_id` 与 `operation_records.task_id` 外键引用（`0007:8-9`），
   重建期间外键是关的 → **这段时间内的任何写都不受约束保护**。
   建议：与其它待发版的 task_type 变更**合并成一次重建**；在发布说明里标注这段窗口；若仓库里 `cache_warmup` 尚未设计，
   先别为它改建表（见 P-06：方案里的 `cache_warmup` 在本仓不存在）。

---

## 完整问题清单（按严重度）

### P-01 【严重】`now >= target` = 补跑风暴，跨天/重启都会多跑
- **现场**：`proposal.md:31-32`；键只到天粒度 `proposal.md:24,29,35`。
- **依据**：排程器现状 `src/worker/scheduled.ts:64-129` 的做法是「**每次启动试着排一次 + 按天键去重**」，
  且明确写了代价（`:11-13`：「一整天没运行就跳过，次日再跑」）。方案改成「每次 tick 都试」，把代价从 O(1) 变成 O(tick)。
- **后果**：
  - 当天：因为键里带 `today`，同一天只会有一次 enqueue 生效（`service.ts:205-208` 命中即返回旧行），所以**不会当天刷爆**；
  - 跨天：worker 停机 → 启动后立即排入当天那一轮（**本该在 03:00 跑的，12:00 才跑**）；
  - 若哪天 key 改成不带日期（或 `ymd` 用了错误时区，见 P-02），就会退化成「每个 tick 一排」，即 10 秒一次。
- **建议**：`if (now >= target && now - target <= 15 * MINUTE)`，并在排入后把「今天已处理」记进内存（`Map<name, day>`），
  **让 tick 的复杂度与「今天是否已排」无关**；错过就记日志跳过，与 `scheduled.ts:11-13` 的既有取舍保持一致。

### P-02 【严重】时区语义未定义，而数据库里全是 UTC 毫秒
- **现场**：`proposal.md:29`（`ymd(now)`）、`:31`（`atTime(now, s.at)`）、`:24-25`（`'03:00'` 字符串）——**三处都没说时区**。
- **依据**：`src/server/platform/time.ts:5-13`（「数据库中所有时刻为 Unix 纪元起算的 UTC 毫秒」）；
  `src/packages/common/time.ts:2-12`（**明写**：`toLocaleString` 用服务器本地时区，线上是 UTC，「本地全对、线上全错」）；
  `src/server/modules/catalog/discovery.ts:121-124`（现有 `discoveryDailyKey` 的注释：按 Asia/Shanghai 算「哪一天」）；
  部署侧：`docker/compose.yaml:38,53,88` 三处硬编码 `TZ: Asia/Shanghai`，但 **`deploy/dsh-review-worker.service` 里没有任何 TZ 设置**（整个文件 `:1-28`）。
- **后果**：同一份代码在 compose 里是 +08:00、在 systemd 里是宿主时区（`deploy/dsh-review-worker.service:1-28` 无 TZ，VPS 常为 UTC）→
  `'03:00'` 在两个部署形态下相差 8 小时；`ymd(now)` 若用 UTC，则北京时间 00:00-08:00 的 tick 会把「昨天」当「今天」，
  造成**同一天两轮、或漏一天**。这是 `common/time.ts:4-7` 已经踩过的坑，方案原样重踩。
- **建议**：把 `'03:00'` 解析成**「Asia/Shanghai 的 03:00 对应的 UTC 毫秒」**，实现放在 `packages/common/time.ts` 旁边（那里已有钉死时区的 `Intl` 实例，`:12-23`），
  并写测试：给定 `now = 2026-01-01T18:59:00Z`（= 北京 02:59），`target` 应为 `…T19:00:00Z`；
  同时给 `deploy/dsh-review-worker.service` 补 `Environment=TZ=Asia/Shanghai`（与 compose 对齐）。

### P-03 【严重】`scheduled_at` 列与 `idx_tasks_scheduled` 在方案里从未被使用
- **现场**：`proposal.md:15-16` vs `:28-41`（tick 只读内存数组）。
- **依据**：`background_tasks` 的既有列已足够：`available_at`（`migrations/0001_baseline.sql:359`）**就是**「什么时候可以执行」，
  且 `claimWithin` 的选取条件正是 `available_at <= ?`（`src/server/modules/tasks/service.ts:122-131`），排序也带上它；
  现有索引 `idx_tasks_claimable(processing_state, available_at, scheduling_group, id)`（`0007:80`）已经覆盖这条查询。
- **后果**：新增列永远是 NULL（除非未来某处写入），新增索引永远不被选中（没有任何查询以 `scheduled_at` 为条件）；
  而 `background_tasks` 是全站写热点：每次 claim 写 2 表、renew 每 30s 写 2 表、finish 写 2 表（`service.ts:243-266,278-395`）——
  **多一个索引 = 每次 INSERT/UPDATE 多一棵 B-tree 维护**，在 SQLite 单文件上是纯损耗。
- **建议**：删掉 §1。若目标是「任务带排程时间可见」，用现有 `available_at` 即可，零 DDL。

### P-04 【高】「复用主循环节拍」= tick 被长任务阻塞，且空转成本被低估
- **现场**：`proposal.md:20`。
- **依据**：`src/worker/main.ts:170-206`：`runOnce()` 里先 `recoverExpired`、再排程、再 `claim` 最多 4 个、**串行 `await execute(claim)`**（`:199-204`）；
  无任务时才 `sleep(IDLE_SLEEP_MS)`（`:219`），而 `IDLE_SLEEP_MS = RECOVERY_SCAN_INTERVAL_MS = 10s`（`constants.ts:25,48`）。
- **后果**：
  - 空转：每天 8640 次 tick，每次至少一次 `recoverExpired` 的 `SELECT`（`service.ts:404-409`，走 `idx_tasks_lease`，很便宜但不为零）；
  - 阻塞：一个跑满 `TASK_DEADLINE_MS = 300s` 的任务会让后面所有 tick 延后最多 5 分钟——**03:00 与 04:00 的 schedule 若前一晚任务堆积，会连环推迟**；
  - 叠加：方案没提「tick 与 `ensureDailyDiscoveryTask` 的关系」，若两者都留着，会**每天排两次发现任务**（新旧两套机制并存）。
- **建议**：tick 用独立的轻量定时器（`setInterval`，不走 `runOnce` 的串行路径），且**排程只做「查表 + enqueue」两次调用**；
  并明确「`src/worker/scheduled.ts` 的旧机制本次删除」，否则两套排程会重复触发（见 P-08）。

### P-05 【高】03:00 与每日备份撞点（IO 与写锁竞争）
- **现场**：`proposal.md:24`（`daily_discovery` @ 03:00）。
- **依据**：`deploy/dsh-review-backup.timer:12-13`（03:00 + 5 分钟随机）；`deploy/dsh-review-backup.service:26`（`backup.mjs --keep 14`，
  读整库 + integrity + 清单）；`docker/compose.yaml:92` 数据目录挂在同一卷上。
- **后果**：备份在读一个正在被高频写入的库（发现任务每收录一个插件就写一次 `operation_records` + `plugins`，
  `src/server/modules/catalog/discovery.ts:269-282`），integrity check 时间与写锁等待都会上升；
  若备份在 03:00 抢到写锁做 checkpoint，发现任务的写入会 `SQLITE_BUSY` 或变慢，反过来拖住 worker 主循环（见 P-04）。
- **建议**：错开（发现放 05:00 或备份改 02:00 后错开 1 小时以上），并像备份那样给自己一个 `RandomizedDelaySec` 语义的抖动。

### P-06 【中】`cache_warmup` 在本仓库不存在，却在 §2/§3 里被当作既有任务类型
- **现场**：`proposal.md:25,44-45`。
- **依据**：`src/packages/common/field-rules.ts:87-96`（`TaskTypes` 只有 5 项，无 `cache_warmup`，且注释明写「**不能由配置注入新类型**」）；
  全仓 grep 无 `cache_warmup` 的处理器（`src/worker/handlers.ts:369-377` 注册表只有 5 个 handler）。
- **后果**：没有 handler 的任务会被判 `skipped`（`src/worker/main.ts:89-104`：`no_handler`），
  即**每天 04:00 白排一个必然 skipped 的任务**，并且每次 `enqueue`→`claim`→`skipped` 都是 2 次写事务（`service.ts:190-226`、`:278-395`）。
  叠加 P-05 的建表重建成本，这个「顺手加的一个例子」成本并不小。
- **建议**：方案里把 `cache_warmup` 标成「本次不实现，仅示例」，或明确列出它的 handler 与依赖；
  在 `TaskTypes`（`field-rules.ts:88-95`）与迁移 CHECK（`0007:26`）**两处同步**（本仓有先例：0007 重建后字段规则也同步了）。

### P-07 【中】新增 task_type 的重建窗口与「单写者」的关系没交代
- **现场**：`proposal.md:44-45`。
- **依据**：`migrations/0007_task_type_plugin_discovery.sql:1-19`、`:62-81`；`src/server/platform/migrate.ts:46-47`（「中途失败会留下半成品」）；
  `docker/compose.yaml:81-83`（worker `depends_on: migrate: service_completed_successfully`）。
- **后果**：重建要对全表 `INSERT SELECT` + `DROP` + `RENAME` + 重建 2 个索引；
  `background_tasks` 行数随时间增长（每天至少 1 行发现任务 + 用户任务），重建耗时与之成正比；
  这段时间 `PRAGMA foreign_keys=OFF`（`0007:19`），任何并发写都不受外键保护；迁移慢 → worker 起得慢 → 全站任务处理延后。
- **建议**：与其它 task_type 变更合并成一次重建；选在备份之外的低峰窗口执行；
  并在方案里注明「本次迁移需要 `dsh:migration-transaction=off`」（`migrate.ts:49` 的标记），否则会被执行器套事务导致 PRAGMA 空操作。

### P-08 【中】新旧两套排程机制并存：同一天排两次发现任务
- **现场**：`proposal.md:20-41` 新增 tick，但**没有说要替换** `src/worker/scheduled.ts`。
- **依据**：`src/worker/main.ts:178-188`（每 `DISCOVERY_SCHEDULE_CHECK_MS = 60s` 调一次 `ensureDailyDiscoveryTask`，`scheduled.ts:28`）；
  新方案的键是 `discovery:github:${d}`（`proposal.md:24`），旧方案是 `daily:${d}`（`discovery.ts:121-124`）——**两个键不同**！
- **后果**：同一天两条不同键的任务都会入队 → 发现任务**跑两次**，配额翻倍（未认证 60/h 下，两轮 ≈ 52 次核心请求，见摘要 1）；
  且旧 key 的行会永久留在 `background_tasks` 里，后台 `GET /api/admin/discovery`（`src/app/api/admin/discovery/route.ts:29-37`，取最近 10 条）会被旧格式淹没。
- **建议**：方案必须写明「删除 `ensureDailyDiscoveryTask` / 保留但复用同一个 key」，并在发布日做一次 key 迁移说明（旧 `daily:` 行保留不动）。

### P-09 【低】没有「今天排了几轮、有没有错过窗口」的可观测
- **现场**：`proposal.md:47-50`「预期」三条都是语义描述。
- **依据**：现有可观测入口是 `scheduled.ts:125-127` 的 `log.info('已排入当天的插件自动发现任务')` 与任务 `progress`；
  新方案把判断逻辑搬进 tick 后，**「为什么今天没排」这条信息会从一行日志退化成没有任何输出**。
- **后果**：错过窗口、时区算错、key 撞车这些故障都表现为「今天没跑」，运维只能翻 journal 逐 tick 看。
- **建议**：tick 里对「跳过（窗口外）/ 已排 / 新排」各打一次结构化日志（每天最多 3 行，成本为零）；
  并在方案里给出一个「调度状态」查询口径（例如从 `background_tasks` 里按 `deduplication_key` 前缀取当日行）。

---

## 结论与建议

**方案在性能/运维维度不可按原样落地**：核心问题是「到点判断」写成了「过了点就补」，而复用主循环又把它绑在一个会被长任务阻塞的 10 秒节拍上；
同时新加的 `scheduled_at` 列与索引在方案自己的代码里从未被读取——**既增加写放大，又没有任何收益**。

**建议的最低改造**（不需要新列、不需要新索引）：
1. **用 `available_at` 表达到点时刻**（`0001_baseline.sql:359`，已是 claim 排序键），调度器只负责「提前把这一天的任务按 target 排进去」；
2. **加触发窗口**：`now - target <= 15 * MINUTE` 才排，窗口外记日志跳过——避免重启/停机后的补跑风暴；
3. **时区钉死 Asia/Shanghai 并写测试**，同时给 `deploy/dsh-review-worker.service` 补 `TZ`（与 `docker/compose.yaml:88` 对齐）；
4. **错开 03:00 备份**（`deploy/dsh-review-backup.timer:12`）；
5. **明确删除或合并旧排程**（`src/worker/scheduled.ts`），否则两套键会各跑一次；
6. `cache_warmup` 要么删掉这个例子，要么补齐 handler + `field-rules.ts:88-95` + 迁移 CHECK 三处。

A 这样改之后：**零新增列、零新增索引、零补跑风暴，调度误差 ≤ 15 分钟窗口内的 tick 间隔**，且与既有备份/迁移/时区纪律一致。

---

> 评审人声明：本报告只覆盖性能/容量/时序/运维成本视角。`schedules` 是否允许动态配置（潜在任务注入面）、
> 权限与审计合规由其他视角负责，本文不代替它们的结论。
> 前提核实：方案「背景」一节（`proposal.md:4-5`）与仓库一致——`ensureDailyDiscoveryTask` 确为「启动即试排 + 按天键」
>（`src/worker/scheduled.ts:64-129`），前提成立；但 §3 里的 `cache_warmup` 在仓库中**并不存在**（`field-rules.ts:87-96`）。
