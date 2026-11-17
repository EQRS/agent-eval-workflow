# m12-schedule-timing · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_题库全测_验证单元_0009
> 模型 deepseek-chat · 11000 字 · 21 次工具调用

---

迁移器按文件名去重并排序执行——这确认了方案中「迁移 0042」会与现有 `0042_audit_immutable.sql` 的版本记录冲突（同一 version 名已在 schema_migrations 里）。

我已掌握足够信息，开始评审。

---

# SRE 评审：m12-schedule-timing「定时任务调度时间改造」

## 结论：**不建议按当前方案上线**。有 1 个阻塞项（迁移号冲突，会直接导致生产迁移失败）、2 个高危项（幂等键设计会让"固定时刻"根本做不到、以及重复执行缺口），其余为中等。

---

### ① 迁移号 0042 已被占用 —— 阻塞，会导致生产迁移直接失败

**问题部分**：方案 §1 写「给 background_tasks 加一列（迁移 0042）」。

**后果**：`migrations/` 下已存在 `0042_audit_immutable.sql`（审计表不可变触发器），且它已记录在 `schema_migrations` 里。迁移执行器 `migrate.ts:88-95` 以**文件名**为 version 主键做去重（`ensureRegistry` 定义 `version TEXT PRIMARY KEY`，见 `migrate.ts:28-31`；`done.has(name)` 见 `migrate.ts:89`）。因此：
- 若新文件也命名 `0042_*.sql`，`listMigrationFiles` 排序（`migrate.ts:50-54`）会让两个 0042 相邻，但只有**文件全名**不同才不会被当成同一个 version——也就是说，命名成 `0042_schedule_timing.sql` 不会硬报错，但**编号语义已经错乱**：线上库的 schema 版本号（`health.ts` 的 `schemaVersion` 取自 `ORDER BY version DESC LIMIT 1`）会变得不可解释，且 0042 之后出现新 0042 会让人误判「这条已经跑过」。
- 更危险的分支：若有人图省事复用了同一个文件名 `0042_audit_immutable.sql` 去改，`done.has(name)` 为 true → **静默跳过，列根本不会被加上**（`migrate.ts:89`），部署"成功"、运行时 `no such column: scheduled_at` 全线爆。


**建议**：改用新号 **0043_schedule_timing.sql**（目录里 0042 已是末尾）。同时在 PR 描述里写明「0042 已属于审计不可变修复」，避免后续再撞。上线前用 `migrate --dry-run` 形态（或先跑 `selfcheck`）确认 `applied` 列表里出现的是 0043。

---

### ② 只加 `scheduled_at` 列，但没有说明 CHECK/执行权不变量 —— 高危

**问题部分**：方案 §1 的建表语句只写 `ADD COLUMN scheduled_at INTEGER;` 和一个索引。

**后果**：`background_tasks` 上一条硬不变量是：
```
running 时 lease_owner / lease_expires_at / execution_token 全非空；
非 running 时三者全空
```
见 `migrations/0001_baseline.sql:368-376`。另有一条已存在的 `started_at >= 0` 风格约束（同文件 360-367 行，`available_at INTEGER NOT NULL CHECK (available_at >= 0)`）。方案的 `scheduled_at` 是裸 INTEGER，没有 `CHECK (scheduled_at IS NULL OR scheduled_at >= 0)`，与全站「时间一律 UTC 毫秒且非负」的约定（`0001_baseline.sql:1-12` 的 TS 约定、`field-rules.ts` 注释）不一致。运维侧的影响是真切存在的：一旦有脏写入（负数或 1970 前的怪值）落库，**没有任何约束拦它**，而调度查询会以它为排序键 → 该任务永远排在最前或被永久跳过，且事后无法用 CHECK 兜底排查。

**建议**：
1. 列定义写全：`scheduled_at INTEGER CHECK (scheduled_at IS NULL OR scheduled_at >= 0)`。
2. 明确语义二选一并在迁移注释里写死：**「计划时刻」**（记录业务意图，不参与领取）还是 **「可执行时刻」**（那就应该复用已有 `available_at`，不要新增列）。**强烈建议走前者**，因为 `available_at` 已经被 `idx_tasks_claimable`（`0001_baseline.sql:378`）和 claim 的 `WHERE available_at <= ?`（`service.ts:126`）使用，混用会直接破坏领取排序。

---

### ③ 方案 §2 的 `tick` 与「固定时刻」的目标自相矛盾：幂等键按"天"去重，`now >= target` 永远真 —— 高危

**问题部分**：方案 §2 代码 `if (now >= target) { enqueue({ deduplicationKey: s.dedup(today) ... }) }`。

**后果**：`enqueue` 的实现是「先 `SELECT id FROM background_tasks WHERE task_type=? AND deduplication_key=?`，存在则直接返回 `created:false`」（`service.ts` 的 enqueue 段，约 176-183 行），**不会更新 `available_at` 也不会复活已终态的任务**。由此产生两个线上问题：

- **不是"固定时刻执行"，而是"启动时刻执行"**：`now >= target` 在 03:00 之后的任何时刻都为真。worker 09:00 启动 → 任务 09:00 才入队 → 09:00 执行。方案的"预期：任务在每天固定时刻执行"**不成立**。这正是产品要修的 bug，方案没修掉，只是换了个写法。
- **错过即永久丢失**：当天 key `discovery:github:2025-xx-xx` 一旦被消费（`succeeded`），当天的行就永远是 succeeded，`UNIQUE(task_type, deduplication_key)`（`0001_baseline.sql:367`）保证不会重建。worker 若 03:00 没活着，当天这次就彻底没了——`src/worker/scheduled.ts:1-20` 的头部注释已经如实承认了这个代价（「如果一整天 worker 都没运行，那一天的发现就跳过了」），方案没有改进它。
- **补偿行为不一致**：现有 `ensureDailyDiscoveryTask` 有 `DISCOVERY_SCHEDULE_CHECK_MS = 60_000` 的节流（`scheduled.ts:32`）以及 env 硬刹车（`DSH_REVIEW_CATALOG_OFFLINE` / `DSH_REVIEW_DISCOVERY_DAILY=0`，`scheduled.ts:46-55`）。方案新写的 `tick` 是**裸循环、每秒每 tick 都可能跑一遍两个 schedule**（方案没有节流），且**完全丢掉了这两个环境变量刹车**——运维出事时最常用的"一行 env 停掉"手段被静默移除了。

**建议**：
1. 保留 `scheduled_at` 但**让它在入队时就等于当天的目标时刻**，并让 worker 的 `available_at` 取 `max(now, scheduled_at)`，这样语义才自洽；而"是否该排今天这条"的判断改成 `now >= target && now - target <= 允许窗口（如 2h）`，超窗就**记日志跳过**而不是补排一个"看起来是 03:00 其实 09:00 才跑"的假任务。
2. 新 `tick` **必须**继承现有节流常量（复用 `DISCOVERY_SCHEDULE_CHECK_MS`）与 `discoveryScheduleSkipReason` 的 env 刹车，不要另起一套。
3. 若产品要"固定时刻"，正确做法是：`available_at = 目标时刻`（未来时刻入队，到点才可领），而不是 `now >= target` 才入队。SQLite + `WHERE available_at <= ?` 的 claim 已经天然支持"未来任务到点才被领"（`service.ts:126`），不需要新机制。

---

### ④ `cache_warmup` 放开 CHECK 需要重建表，方案漏掉了这个"高危操作"的全部运维前提 —— 高危

**问题部分**：方案 §3「cache_warmup 需要新的 task_type（迁移里放开 CHECK）」。

**后果**：`task_type` 的 CHECK 是**列内 CHECK**，SQLite 无法 `ALTER`（`0007_task_type_plugin_discovery.sql:5-8` 已把这个坑写清楚）。放开枚举**必须重建整张 `background_tasks`**，而它被 `background_task_attempts.task_id` 与 `operation_records.task_id` 外键引用（`0001_baseline.sql` 的 `background_task_attempts` 定义处 `task_id TEXT NOT NULL REFERENCES background_tasks(id)`）。因此这条迁移**必须**：
- 文件头声明 `-- dsh:migration-transaction=off`（`migrate.ts:44-56`），自己写 `PRAGMA foreign_keys = OFF; BEGIN; ... COMMIT; PRAGMA foreign_keys = ON;`
- **自己保证原子性**——执行器不会替你回滚，中途失败会留下半成品表（`migrate.ts:46-52` 明确警告）。
- 重建后**手工重建索引**；方案 §1 的 `idx_tasks_scheduled` 若在重建前建，会随 `DROP TABLE` 一起消失。

这是本次改造**风险最高的一步**（线上任务队列整表重建），而方案把它压缩成一句话"迁移里放开 CHECK"。方案也未提 `TaskTypes` 常量需要同步（`src/packages/common/field-rules.ts` 的 `TaskTypes` 数组，注释明确写「注册集合随迁移版本维护，不能由配置注入新类型」）——只改 SQL 不改常量，`enqueue` 的 `taskType` 类型检查不过，或反之改常量不改 SQL → 落库被 CHECK 拒。

**建议**：
1. 迁移文件里照抄 0007 的骨架（`0. 文件头 marker`、1. `PRAGMA foreign_keys=OFF`、2. `BEGIN`、3. 建 `_new` 表**含新 CHECK + scheduled_at**、4. **逐一列名列值**拷贝（不要 `SELECT *`，0007 注释明说理由）、5. `DROP`、6. `RENAME`、7. **重建全部四个索引**：`idx_tasks_claimable`、`idx_tasks_lease`、`idx_tasks_scheduled`（+ 任何后续新增的）、8. `COMMIT`、9. `PRAGMA foreign_keys=ON`）。
2. **一次重建同时做完"加列 + 放开 CHECK + 建索引"**，不要拆成两个都需重建表的迁移（省一次整表重建风险窗口）。
3. 上线前**演练并在演练库上跑 `PRAGMA foreign_key_check`**（`migrate.ts` 末尾已内置这个自检并会把不一致计数写进 problems，见该文件末段）。
4. 迁移前**强制备份 DB**，并确认 `restore-backup.mjs` 可用于回滚——重建表是少数"回滚只能靠备份"的操作。
5. `field-rules.ts` 的 `TaskTypes` 与 SQL CHECK **必须在同一个 PR** 里改，并在测试里加一条"常量集合 == SQL CHECK 集合"的防漂移用例（`scheduled.ts` 注释提到过 payload 键名漂移踩过线上坑，同类问题）。

---

### ⑤ 调度循环放在 worker 内，但 worker 是单实例假设 —— 需要明确并加护栏

**问题部分**：方案 §2「worker 里跑一个轻量调度循环（复用现有主循环的节拍）」。

**后果**：`tick` 里没有跨进程互斥。虽然 `UNIQUE(task_type, deduplication_key)` 让重复 enqueue 幂等（`0001_baseline.sql:367`，`service.ts` enqueue 先查后插），但 `enqueue` 是 **先 `SELECT` 再 `INSERT`**（非 `INSERT ... ON CONFLICT`），两个 worker 同时 tick 会形成 SELECT 都空 → 双 INSERT 的竞态窗口；好在 UNIQUE 会挡住第二条并抛 `SQLITE_CONSTRAINT`。而 `main.ts` 里排程调用是包在 try/catch 里、"排程失败不能拖垮主循环"（`main.ts` runOnce 注释），所以**不会崩，但会天天刷一条 error 日志**，长期掩盖真实故障。运维最怕的正是"天天报错、于是没人看告警"。

**建议**：
1. 在迁移注释与部署文档里写明**「本调度器假设单 worker 实例」**，并在运维手册标注：扩容到多 worker 前必须先给调度加互斥。
2. 或者干脆把 enqueue 改成 `INSERT ... ON CONFLICT(task_type, deduplication_key) DO NOTHING`，从根上消掉竞态（需要 `service.ts` enqueue 段配合改，改动小）。
3. 日志分级：约束冲突属于预期内竞态，打 `debug`/`info`，不要打 `error`。

---

### ⑥ 新增 schedule 缺少"卡片可见性"：`scheduling_group` 是必填 NOT NULL，方案代码根本没传

**问题部分**：方案 §2 的 `tasks.enqueue({ taskType, deduplicationKey, requestedBy, payload })`。

**后果**：`scheduling_group TEXT NOT NULL CHECK (trim(scheduling_group) <> '')`（`0001_baseline.sql:356`），且服务层签名里 `schedulingGroup: string` 是**必填**（`service.ts` 的 `TasksService.enqueue` 接口定义处）。方案示例代码**漏了 `schedulingGroup` 和 `availableAt`**（`availableAt` 也是必填，见同一接口）。照抄方案会直接编译失败或运行时 NOT NULL 失败。这不是纯代码问题：`DISCOVERY_SCHEDULING_GROUP = 'plugin_discovery_schedule'` 的存在理由是「与用户任务分开，便于观测系统调度占了多少」（`scheduled.ts:35-36`）——**这是运维观测手段**。新增 `cache_warmup` 若不复用/不新增独立 group，运维在后台就分不清哪些 Task 是定时排的，容量规划失去依据。

**建议**：示例代码补全 `schedulingGroup`（建议新增 `'cache_warmup_schedule'` 或复用统一 `'system_schedule'`，并更新注释说明与用户任务隔离）与 `availableAt`（应为计算出的目标时刻，而非 `now`）。上线前在后台确认能按 group 过滤出这两类任务。

---

### ⑦ 缺少 `atTime` / `ymd` 的时区声明，运维排查时会踩"少 8 小时"

**问题部分**：方案的 `atTime(now, '03:00')` 与 `ymd(now)` 未定义，也未写时区。

**后果**：全站约定时间是 **UTC 毫秒**（`0001_baseline.sql:1-12`、`0004_runtime_heartbeats.sql` 注释「时间一律 UTC 毫秒」）。`'03:00'` 究竟指 UTC 03:00 还是本地 03:00，方案没写。后果是线上跑在 03:00 UTC = 11:00 北京时间，或反之；而 dedup key 用 `ymd(now)` 取"今天"，**若按本地时区算 ymd 而按 UTC 算 atTime**，会在时区边界（如 UTC+8 的 00:00-08:00 时段）出现"key 是明天、时刻却是昨天"的错位，导致**同一天跑两次或直接漏跑**，且日志上极难看出来。

**建议**：在方案里明确写死——`atTime` 按 UTC 解释、`ymd` 按 UTC 取日，并在日志里同时打印 `now`(ISO/UTC)、`target`(ISO/UTC)、`today` 三个值，便于事后对账。如果要按业务本地时区，就必须在配置里显式声明 `TZ offset` 并写进 dedup key，不能靠进程环境 `TZ`（容器里通常就是 UTC，本地开发机可能是 +8，环境不一致会让问题只在生产暴露）。

---

### ⑧ 未提可观测性：定时任务失败后运维无从判断

**问题部分**：方案的「预期」只有三条功能向描述。

**后果**：现有系统已有心跳表 `runtime_heartbeats`（`0004_runtime_heartbeats.sql`）与 `health.ts` 自检，方案没说新增 schedule 是否纳入。若 `cache_warmup` 连续几天失败，任务会走 `MAX_RECOVERABLE_ATTEMPTS = 3` + `RETRY_BACKOFF_MS = [30s,120s]` 进终态 `failed`（`constants.ts:34-45`），**此后当天不再有任何重试**，而运维只有在后台页翻任务列表才能发现。定时任务的典型故障模式恰恰是"静默不跑"。

**建议**：
1. 在 `runtime_heartbeats.detail` 里带上**每个 schedule 的 last_fire_at / last_result**（`detail` 是 JSON 且允许任意上下文，见 `0004_runtime_heartbeats.sql:19`），让健康检查能一眼看出"最后一次 03:00 排入是什么时候"。
2. 对「目标时刻已过但当天 key 不存在」的情况打 `warn` 级日志并计入自检 problems（`health.ts` 已有 problems 汇总机制），这样"漏跑"能被告警而不是靠人盯。
3. 明确 `cache_warmup` 失败的运维处置路径（是否有 env 硬刹车，如 `DSH_REVIEW_CACHE_WARMUP=0`），与 `DSH_REVIEW_DISCOVERY_DAILY=0` 对齐。

---

### ⑨ 与现有 `ensureDailyDiscoveryTask` 的关系未交代 —— 有双跑风险

**问题部分**：方案要把 `daily_discovery` 搬进新 `schedules` 数组（`dedup: (d) => 'discovery:github:${d}'`）。

**后果**：现有链路是 `main.ts` 的 `runOnce()` 在 `scheduledTasks === true` 且过 `nextScheduleCheckAt` 节流时调 `ensureDailyDiscoveryTask`（`main.ts` 的 runOnce 段）。方案**没有说旧调用点是删除还是保留**。如果保留且新 `tick` 同时上线：dedup key 若字面完全一致（`discovery:github:${ymd}` vs 现有 `discoveryDailyKey(...)`），UNIQUE 会挡住，`created:false`，**表面安全**；但两处的 `payload` 不同（现有 payload 有 `run_key / query / search_pages / per_page / max_candidates / max_plugins / max_requests`，`scheduled.ts` 中段），先跑的那条赢，**另一条被静默丢弃**——若新 `tick` 的 payload 更"简陋"（方案示例只有 `{date}`），就可能让发现任务按错误的预算运行。`scheduled.ts` 的中段注释明确记录过这类事故：「曾经这里写成 camelCase，结果 handler 读不到 → 预算被静默忽略（线上第一轮用了 60 个候选，而不是预算里的 20）」。

**建议**：在同一 PR 里**显式删除**（或明确 gate 掉）`main.ts` 对 `ensureDailyDiscoveryTask` 的调用，只保留一个排程入口；并且新 `tick` 复用 `ensureDailyDiscoveryTask`（或至少复用它的 payload 构造与 `discoveryBudgetFor`），**不要重写一份 payload**。payload 键名必须与 `handlers.ts` 读取名一致。

---

### ⑩ 小项：`at: '04:00'` 与 03:00 的 cache_warmup 无预算/节流，可能和发现任务抢 SQLite 写锁

**问题部分**：`schedules` 里 `cache_warmup` 04:00、`daily_discovery` 03:00。

**后果**：better-sqlite3 是同步驱动、单写者（`main.ts` 注释「本切片按顺序执行（better-sqlite3 是同步驱动）」）。两个重量级任务若时间接近（或发现任务因重试拖到 04:00，重试等待 30s/120s，见 `constants.ts:52`），会与用户任务的 claim 争抢写锁，表现为 `SQLITE_BUSY`/延迟抖动。方案没写 `cache_warmup` 的规模上限。

**建议**：错开间隔（≥1h 且不落在重试窗口内），给 `cache_warmup` 也加预算上限（对齐 `discoveryBudgetFor` 的思路）与 env 刹车，并在后台配置页可调。

---

## 上线前 SRE checklist（按方案当前形态）

| # | 项 | 状态 |
|---|---|---|
| 1 | 迁移号改为 0043（0042 已被 `0042_audit_immutable.sql` 占用，见 `migrate.ts:88-95`） | ❌ 必须改 |
| 2 | `dsh:migration-transaction=off` + 自管事务 + 重建全部索引（`0007` 骨架） | ❌ 方案缺失 |
| 3 | 迁移前整库备份 + 演练库跑通 `foreign_key_check` | ❌ 方案缺失 |
| 4 | `field-rules.ts` `TaskTypes` 与 SQL CHECK 同 PR 修改 + 防漂移用例 | ❌ 方案缺失 |
| 5 | 旧 `ensureDailyDiscoveryTask` 调用点显式下线，避免双排程 | ❌ 未交代 |
| 6 | `scheduled_at` 加 `>= 0` CHECK，语义写明（意图时刻 vs 可执行时刻） | ❌ 方案缺失 |
| 7 | 保留 `DSH_REVIEW_CATALOG_OFFLINE` / `DSH_REVIEW_DISCOVERY_DAILY` 刹车；新任务也要有自己的刹车 | ❌ 方案丢失 |
| 8 | 时区写死（UTC）+ 日志打印 now/target/today | ❌ 未声明 |
| 9 | 心跳 detail 记录各 schedule 的 last_fire_at，纳入自检 problems | ❌ 方案缺失 |
| 10 | `schedulingGroup` / `availableAt` 补全（NOT NULL 且是运维观测维度） | ❌ 示例代码漏 |

**最核心的一句**：方案想解决「执行时刻不稳定」，但 §2 的 `now >= target` 让执行时刻**仍然等于 worker 启动/循环时刻**，只是把 `ensureDailyDiscoveryTask` 的逻辑重写了一遍并丢掉了它的刹车和节流。真正要固定时刻，请走 `available_at = 目标时刻`（未来入队、到点才可领），claim 的 `WHERE available_at <= ?` 已经现成支持。