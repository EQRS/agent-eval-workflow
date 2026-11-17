# n71-admin-split · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0031
> 模型 deepseek-chat · 6587 字 · 23 次工具调用

---

# SRE 评审 · 方案《管理后台改造成独立应用》

**结论：方案不可行，不能进入实施。** 提案只有 19 行，但它的三条设计恰好击中了这个系统现有的三条 SRE 不变量：单库单写者、单进程连接单例、Cookie/会话单一信任域。下面逐条说明。

评审依据的实物：`migrations/0001~0042`（40 份）、`src/server/platform/{db,health,heartbeat,migrate,session-cookie,file-store,rate-limit}.ts`、`src/worker/main.ts`。

---

## 1. 「共享数据库：两个应用连同一个 SQLite 文件」— 最严重，必须否决

**① 哪部分有问题**：`proposal.md:9-10`「两个应用连同一个 SQLite 文件」。

**② 后果（附代码/迁移证据）**

- `src/server/platform/db.ts:56` 硬编码 `journal_mode = WAL` + `db.ts:57` `synchronous = FULL`。WAL 是**单写者**模型：任一时刻只有 1 个进程能持有写锁。拆成两个进程后，写竞争从「进程内串行」变成「跨进程抢锁」。
- `db.ts:60` `busy_timeout = 5000`（默认 5 秒）。而且 `db.ts:8-10` 的注释自己已经写明：「better-sqlite3 是同步驱动，等待会阻塞所属线程」。也就是说，**锁等待 5 秒 = 该进程整个事件循环停摆 5 秒**，主站和管理后台互相拖死，而不是只慢一个接口。
- 后台的负载恰好全是最重的写：`migrations/0042_audit_immutable.sql:17-27` 给 `operation_records` 加了不可删不可改守卫，说明全站审计与幂等去重都挤在这一张表上；`migrations/0040_user_roles.sql` 的 `user_roles/role_applications`、`0037/0038` 的论坛审核、`0031 batch_jobs` 导入批次，全是后台写、主站读。写入方从 1 个变 2 个，`SQLITE_BUSY` 会成为常态而非异常。
- **最致命的一点**：`db.ts:74` `getDb()` 是**进程级单例**，而 `migrations/0004` + `src/server/platform/heartbeat.ts:19-25` 的健康判定是「`runtime_heartbeats` 里 `name='worker'` 那一行的心跳年龄」。两个应用若共用这份 DB，两个进程会写**同一个** `name='worker'` 行（`heartbeat.ts:19` 是常量），并且 `heartbeat.ts:104` 的 `beginRun` 会**重置 `started_at`**。结果是双方互相覆盖对方的心跳 —— 监控面板上看一切正常，实际任一进程早已卡死。提案没有定义第二个心跳名，`health.ts` 的 `worker.heartbeat` 检查会退化成「两者有一个活着就算健康」。
- 提案没提备份/恢复。`0042` 的注释里明确点出已有一条「`restore-backup.mjs --force` 整库覆盖」加「删审计行」的伪造通道。整库覆盖是**对两个应用同时生效**：任何一方做恢复演练，都会把另一方的在途事务和会话一起冲掉。

**③ 建议**

- **否决「共享同一个 SQLite 文件」。** 两个应用必须是**各自独立的库**，后台通过 API 调主站（后台不直连主站库）。
- 若因工期必须共享，则最低门槛（缺一不可，且需 SRE 签字）：
  1. 明确**唯一写者**：后台只写自己的 `admin_*` 前缀表，主站表只读；或后台所有写走主站内部 API。
  2. 心跳按进程分名，例如 `WORKER_HEARTBEAT_NAME`（`heartbeat.ts:19`）改成 `worker:main` / `worker:admin`，`health.ts` 分项检查、分项 503。
  3. 给出 `busy_timeout` 的容量测算（当前 5000ms 是按单写者调的），并明确写入 p99 的验收线。
  4. 写清**备份必须是双进程一致的同一时间点**（WAL 下 `app.sqlite` + `-wal` + `-shm` 三件套一起取，不能只拷主文件）。

---

## 2. 「后台应用自己实现登录，复用 users 表」— 认证面被翻倍，且与现有 Cookie 约束冲突

**① 哪部分有问题**：`proposal.md:12-13`。

**② 后果**

- `src/server/platform/session-cookie.ts:20-22,56-73` 定义的是**单个信任域**：生产用 `__Host-dsh_session`，前缀规则要求 Secure + Path=/ + **无 Domain**。`__Host-` 在规范上禁止 Domain 属性，所以 `__Host-dsh_session` **无法天然跨 `admin.example.com` 与 `example.com` 共享**。提案没写后台用什么 cookie 策略，实施时必然二选一：要么放宽成带 Domain 的 cookie（等于放弃 `__Host-` 的防子域注入保护），要么后台另起一套会话表 —— 前者是安全回退，后者会导致 `user_sessions`（`0001_baseline.sql:47-63`，含 `absolute_expires_at` 与 `token_digest` 长度 64 的 CHECK）被两套代码各自读写。
- `0001_baseline.sql:22` `users.role CHECK (role IN ('user','admin'))`，`0040_user_roles.sql:9-13` 明确区分「权限底座 `users.role`」与「叠加身份」。后台「自己实现登录」意味着出现了**第二处判定 `role='admin'` 的代码点**。`0042` 的审计不可变守卫保护的是 `operation_records`，但 `0002_triggers.sql` 的 33 个触发器保护的是业务表 —— 如果后台应用绕过主站模块直接写库，这些触发器仍在，但**服务层的权限/幂等逻辑会被跳过**，`operation_records` 的 `UNIQUE(action_type, operation_key)` 去重语义（`0042:6-9` 明确列为 setUserRole/setUserStatus 的关键依赖）会失效。
- `session-cookie.ts:33-47` 的 `DSH_REVIEW_INSECURE_COOKIES` 逃生口会变成**两个应用各有一份**。上线时若只在一个应用上漏删，等于在另一个域名上开了明文会话 —— 这种问题靠人记是记不住的（代码注释里已经写了「线上实测踩到过」）。
- 登出/封禁/改密不再是单点操作：`users.account_status` 与 `management_version`（`0001:20-21`）的失效广播需要跨两个应用，提案零字未提。

**③ 建议**

- 后台登录**不要重写**，改为对接主站现有的 OAuth + `user_sessions` 体系（唯一登录实现），后台只做「有 admin 角色才放行」的鉴权。
- 若必须独立会话，明确写出：① 后台 cookie 名与策略（建议也走 `__Host-`，即后台只在 `admin.example.com` 上持有自己的会话，不做跨域共享）；② 封禁/登出的失效路径与 SLA（要求 ≤1 个 `management_version` 递增周期）；③ `operation_records` 的写入必须由后台**自己的**模块执行，禁止裸 SQL。

---

## 3. 迁移与部署：提案默认「DDL 是幂等无害的」，但现有执行器不允许两个进程并发迁移

**① 哪部分有问题**：`proposal.md:6-7`「后台路由迁到 admin.example.com，独立构建」—— 隐含两个应用各自启动、各自连库，但没写谁跑迁移。

**② 后果**

- `src/server/platform/migrate.ts:60-64,76-120` 是「读 `schema_migrations` → 逐个文件在独立事务里 `db.exec` + `INSERT`」。这个实现在**单进程启动**下是对的；两个进程同时启动时，两者都会读到「未应用」，然后同时尝试 `CREATE TABLE`，后到者拿到 `SQLITE_BUSY` 或「table already exists」，启动直接失败。`src/instrumentation-node.ts:20-39` 的启动自检**刻意不退出进程**（注释写得很清楚：「库没迁完时反复重启只会更难查」），于是结果是**进程活着、`/api/health` 返回 503、编排器却不重启**——比直接崩掉更难运维的「僵尸部署」。
- `migrate.ts:32-49` 的 `NO_TRANSACTION_MARKER` 逃生口（目前 `0007` 在用）是**执行器不替它回滚**的。两个进程并发跑 `0007` 这类重建表迁移，可能留下半成品表，而且 `migrate.ts:104-110` 只能把「执行成功但登记失败」变成响亮错误，无法回滚。
- 反向风险：提案说后台「独立构建」，若后台自带一份 `migrations/` 副本，两份会**漂移**。`migrate.ts:41` 按文件名去重，两份目录的来源不同就无从判断实际应用了哪个版本 —— `health.ts:79` 的 `schema_version` 会变成一个不可信的数字。

**③ 建议**

- 明文规定：**迁移只由主站发布流水线执行一次**，后台容器启动时**禁止**执行 `migrate()`，只允许 `selfCheck()` 并在不合格时 `exit 1`（与现在 `instrumentation-node.ts` 的「不退出」相反，这里应该退出，因为重试是有意义的）。
- 后台 `migrations/` 不要复制，改为同一构建产物中的共享路径或单一来源镜像。
- 若确认要保留双跑，则必须加锁：在 `migrate()` 外层用 `BEGIN IMMEDIATE` 或独立 lock 文件，并把 `busy_timeout` 在迁移场景调高。

---

## 4. 健康检查 / 监控会从「一个信号」退化为「两个都可能骗人」

**① 哪部分有问题**：提案没提可观测性，但拆分改变了 `health.ts` 的语义基础。

**② 后果**

- `health.ts:126-150` 的 `statusForHealth` 只在 `fail`（非 `degraded`）返回 503，设计意图是「别让编排器乱重启」。拆成两个应用后，**主站健康而后台挂**、**后台健康而主站挂** 两种情况需要不同的告警路由，提案没有任何 `/api/health` 契约说明。
- `health.ts:36-47` 的 `requireWorker` 开关（注释：「web 的存活检查不该因为 worker 挂了而被重启」）在双应用下无法区分「哪个 worker」。`heartbeat.ts:22-25` 的 `STALE_MS=30s / DEAD_MS=120s` 阈值是按**单一 worker 的 5s 心跳间隔**（`heartbeat.ts:28`）定的，后台若负载不同（见第 1 条的锁竞争），会频繁误报 stale。
- `health.ts:19-22` 的磁盘检查 `minFreeMb` 默认 200MB，检查的是 `dbFile` 所在分区。两个应用若共用 `.data`，媒体文件（`file-store.ts:161` `defaultMediaRoot()` 也在 `.data` 下）和 DB 抢同一个分区，**后台一次批量导入**（`0031 batch_jobs`）就能把主站的磁盘打到 200MB 以下，两边一起 fail。

**③ 建议**

- 提案必须补一节「可观测性契约」：两个应用各自的 `/api/health` 语义、各自的 `runtime_heartbeats.name`、各自的告警路由与 runbook 链接。
- `minFreeMb` 改为按分区分别配置，且明确「媒体与 DB 是否同盘」——建议拆到不同挂载点。

---

## 5. 回滚与灰度：提案只写了「预期」，没写反向路径

**① 哪部分有问题**：`proposal.md:16-19`「预期：后台独立迭代 / 不影响主站」—— 这两条是**结论**，不是可验收的指标。

**② 后果**

- 「不影响主站」在共享库前提下是**伪命题**（见第 1 条），没有办法用任何 SLO 证明。
- 没有回滚路径：`admin.example.com` 的 DNS/TLS/会话一旦切换，回切意味着后台写的数据（若独立库）要回灌主站 —— 提案没有任何数据归属说明。`0042` 已记录过一次「整库覆盖」造成的伪造通道，同类风险在拆分后会被放大成「两个库的不一致」。

**③ 建议**

- 把「预期」改写成可验收项：① 主站 p99 延迟变化上限；② `SQLITE_BUSY` 错误率上限；③ 后台独立发布时主站零重启；④ 回滚演练步骤与时长（要求一次性完成、可回滚）。
- 明确数据归属：后台产生的数据写在哪个库、属于谁、如何备份、如何销毁。

---

## 优先级汇总

| 级别 | 条目 | 一句话 |
|---|---|---|
| **阻断（Blocker）** | 第 1 条 共享 SQLite 文件 | WAL 单写者 + `busy_timeout=5000` + 同步驱动（`db.ts:8-10,56-60`）= 双进程互相阻塞；心跳名冲突（`heartbeat.ts:19,104`）使监控失真 |
| **阻断（Blocker）** | 第 3 条 迁移归属未定义 | `migrate.ts` 无并发保护 + `instrumentation-node.ts:34-38` 自检失败不退出 = 僵尸部署 |
| **高** | 第 2 条 后台自实现登录 | `__Host-` cookie 无法跨子域（`session-cookie.ts:20-22`）；权限判定出现第二处，绕过 `operation_records` 幂等（`0042:6-9`） |
| **中** | 第 4 条 健康检查语义 | `health.ts` 的 `requireWorker` / `STALE_MS` / `minFreeMb` 均按单进程单盘设计 |
| **中** | 第 5 条 无回滚与验收指标 | 「不影响主站」无法被证伪 |

**我的评审结论：不批准。** 请提案方补齐「数据库归属与写者唯一性」「迁移执行归属」「会话/cookie 策略」「可观测性契约」「回滚方案」五节后重审。其中第 1 条若维持「共享同一 SQLite 文件」，我会直接否决 —— 这不是配置能调好的问题，是架构约束。