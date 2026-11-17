# 评审报告 · n66-warmup-orchestration（SRE 视角）

## 摘要（前 5 条最重要）

1. **【严重】预热失败即 `process.exit(1)`，与两个部署形态的自动重启策略叠成重启风暴** — `deploy/dsh-review-web.service:20-21`（Restart=always / RestartSec=3）、`docker/compose.yaml:68`（restart: unless-stopped）、`docker/docker/web.Dockerfile:76-77`（HEALTHCHECK start-period=30s/interval=30s）× `proposal.md:26-36` — 建议改成「预热失败打 error 日志 + 继续启动，由 /api/health 返回 503 让编排与监控发现」，与 `src/instrumentation-node.ts:30-34` 的既有「刻意不退出」语义一致。

2. **【严重】方案假定的预热对象有三处不存在** — `proposal.md:14/17/20/22` vs 仓库：`db.prepare("SELECT 1").get()` 不在任何现有路径（`src/server/platform/db.ts:44-64` 的 openDatabase 已含 PRAGMA 往返）；`loadSiteSettings()` 全仓不存在（唯一实现是 `src/server/modules/settings/index.ts:343 effectiveSettings(db)`）；`listPlugins()` 是分页查询（`src/server/modules/catalog/service.ts:784`），不是"插件列表"预热，且要传 `PluginDirectoryQuery`；`log` 未 import（`src/server/platform/log.ts:34` 是 `log.info(message, fields)`）。建议按实测调用形态改写，不要凭想象命名。

3. **【严重】重蹈项目已论证过要避免的错误：模块作用域 `process.exit(1)` 会在构建期执行** — 既有护栏 `src/instrumentation-node.ts:11`（`NEXT_PHASE === 'phase-production-build'` 直接 return），而 `proposal.md:30-36` 的 try/catch 在模块顶层 — 建议预热必须放进 `registerNodeRuntime()` 并且受同一 phase 守卫，否则 `next build`（`docker/web.Dockerfile:45` / `package.json build`）可能在镜像构建阶段中途退出。

4. **【中】"预热"在 web 进程里没有收益面，在 worker 进程里是重复劳动** — `server.mjs:45-50`（`app.prepare()` 期间 Next 已开始按需加载与自检）、`src/worker/main.ts:294-298`（worker 不走 instrumentation）、`src/server/platform/db.ts:67-74`（进程级单例，谁先调谁就是预热者）— 建议明确"预热只对 web 生效"或干脆不预热，用冷启动首请求耗时数据（`server.mjs:64-66 duration_ms`）证明收益。

5. **【中】没有超时、没有降级、没有指标：预热挂起会让启动窗口永远停在"既不 healthy 也不 exit"** — `proposal.md:12-23` 三个裸 await × `db.ts:56 busy_timeout=5000` 仅约束单条语句，不含跨语句与锁升级 — 建议每一步带 aborted signal + 总预算（如 5s），超时记 warn 继续（与第 1 条合并成同一个降级策略）。

---

## 二、逐项问题（按严重度）

### 【严重】S1 重启风暴：exit(1) × Restart=always / unless-stopped
- 出问题部分：`proposal.md:26-36` 的失败处理。
- 后果：`deploy/dsh-review-web.service:20-21`（Restart=always, RestartSec=3）→ 库没迁完/盘满时每 3 秒重启一次；`docker/compose.yaml:68`（unless-stopped）→ 无退避地循环重启，容器日志被启动失败淹没（`compose.yaml:70-74` 只保留 10m×3）。`docker/web.Dockerfile:76-77` 的 `start-period=30s` 会被反复重置，编排永远等不到 healthy。
- 建议：预热失败不决定进程生死；退出策略只在"结构性不可用"上做，而现有 `src/server/platform/health.ts:82/97/132` 已经把判据集中在 /api/health（503）。要"启动时发现问题"，用日志 + 健康端点 + 监控告警，而不是 `exit(1)`。

### 【严重】S2 假定的对象/函数不存在（方案与代码基线不符）
| 方案引用 | 仓库实测 | 依据 |
|---|---|---|
| `db.prepare("SELECT 1").get()` | `getDb()` 首次调用时 openDatabase 已跑 `foreign_keys`/`defer_foreign_keys`/`journal_mode`/`synchronous`/`busy_timeout` 五次 PRAGMA 往返，单例缓存 | `src/server/platform/db.ts:44-74` |
| `loadSiteSettings()` | 不存在；等价物是 `effectiveSettings(db)`，一次 `SELECT key,value FROM site_settings` | `src/server/modules/settings/index.ts:343-351` |
| `listPlugins({})` | 是目录分页查询，签名 `listPlugins(req: PluginDirectoryQuery)`，`{}` 等价于"首页第一页" | `src/server/modules/catalog/service.ts:784-829` |
| `log.info("warmup_done")` | `log` 未 import；签名是 `log.info(message, fields)`，字符串 payload 会进 `redact` 前的 fields 位 | `src/server/platform/log.ts:34-38` |
- 建议：方案里每个预热项都要给出**代码坐标**（文件:行号）与**冷启动实测**，否则这一节无法验收。

### 【严重】S3 与既有启动自检冲突，形成"两次启动逻辑"
- 出问题部分：`proposal.md:9-23` 另起一个 `instrumentation.ts` 的 `register()`。
- 后果：`src/instrumentation.ts:18-23` 已有 `register → registerNodeRuntime()`；`src/instrumentation-node.ts:20-37` 已有"打开连接 + selfCheck + 不退出"的启动语义。另写一套等于**两处判断库状态**，且 `selfCheck` 已经做了 60+ 张表的 `sqlite_master` 扫描（`src/server/platform/migrate.ts:249-253`）与 `foreign_key_check` 全库检查（`migrate.ts:262-263`）——这已经是一次"库预热"。
- 建议：预热逻辑并入 `registerNodeRuntime()`，复用同一次 `getDb()` 与 `selfCheck` 结果，不新建启动路径。

### 【中】S4 `listPlugins` 预热 = 冷启动最贵的一次查询
- 出问题部分：`proposal.md:20`。
- 后果：`src/server/modules/catalog/service.ts:137-194` 有 4 个相关子查询（`PUBLIC_REVIEW_COUNT_SQL`/`PUBLIC_ASSESSMENT_TIER_SQL`/`PUBLIC_SECURITY_AUDIT_COUNT_SQL`/`PUBLIC_ASSESSED_COUNT_SQL`），每个都是 `reviews ⋈ review_revisions ⋈ content_entities` 三表关联；`service.ts:788` 的 `COUNT(*)` 还要再跑一遍 `buildDirectoryWhere` 的 WHERE。启动时跑它，等于**把最重的查询塞进启动窗口**，与"首屏变快"目标相反；且预热结果没有缓存，请求到达时照样重跑（`proposal.md:3-5` 的"缓存"没有任何落点）。
- 建议：预热若保留，只做 O(小) 的确定性项（schema 已在校验、设置表一次小查询）；列表查询的优化属于索引/缓存层设计，不该挂启动钩子。

### 【中】S5 没有超时/预算：启动可能永久挂起
- 出问题部分：`proposal.md:12-23`。
- 后果：`db.ts:56` 的 `busy_timeout=5000` 只管单条语句的锁等待；跨语句、迁移进程正在执行 DDL（`docker/compose.yaml:27-42` 的 migrate 一次性进程）、或 `statfs`/NFS 抖动都可能让 await 挂住。此时容器既不 healthy（`web.Dockerfile:76-77` 只在 30s 后开始判）也不退出，编排的 restart 策略救不了它，只能靠人工 `docker restart`。
- 建议：总预算 + 每步 `AbortSignal.timeout`；超时按"降级启动"处理并记 `warmup_timeout` 指标。

### 【中】S6 语义反转：从"带病启动 + 健康端点报错"退回"启动即死"
- 出问题部分：`proposal.md:26-36`。
- 后果：`src/instrumentation-node.ts:30-34` 的注释把这条设计纪律写死了（"库没迁完时反复重启只会更难查；/api/health 会返回 503、容器健康检查与诊断脚本都会报"）。方案把它改回去，等于让**库结构缺失**从"页面 500 + 503 可观测"退化成"进程反复重启、日志被刷屏"。
- 建议：保留带病启动，只把"预热结果"作为 `/api/health?detail=1` 的一项（`src/server/platform/health.ts:20-25` 的 checks 数组扩展一位即可），并纳入 `scripts/diagnose.mjs` 的 problems 列表。

### 【中】S7 与 migrate 进程和启动自检的时序竞争
- 出问题部分：`proposal.md:9-23` 隐式假设库已就绪。
- 后果：`docker/compose.yaml:46-48` 用 `service_completed_successfully` 保证迁移先跑，这条是对的；但 **systemd 部署形态没有这层保护**（`deploy/dsh-review-web.service:4-5` 只有 `After=network-online.target`）——web 可能先于人工执行的 `db:migrate` 启动，此时预热必然失败 → 若采纳 `exit(1)` 就进入 S1 的重启风暴。另外 `deploy/dsh-review-worker.service:8` 的 `After=dsh-review-web.service` 只保证顺序，不保证 web 健康。
- 建议：systemd 形态补 `ExecStartPre=... db:migrate` 或明确的迁移顺序约定；预热项要能区分"暂时不可用（等迁移）"与"结构性损坏"。

### 【中】S8 多进程重复预热 + 观测诉求已有载体
- 出问题部分：`proposal.md:20-22`。
- 后果：web(`server.mjs:45-50` 的 `app.prepare()`) 与 worker(`src/worker/main.ts:294-298`) 都隐含"启动即用库"；web 端 Next 会为多个 server 实例/路由段重复进入 `register()`（Next 16 的 instrumentation 在构建期与运行期、edge/node 两个 runtime 都会被求值，`src/instrumentation.ts:18-23` 已按 `NEXT_RUNTIME` 分流过），预热若不加"只跑一次"守卫（模块级 promise 缓存）就会被调用多次。
- 建议：`const warmupOnce = (() => { let p; return () => (p ??= doWarmup()); })()` 这类幂等守卫，并在日志里打 `pid + runtime`，否则同一进程的重复日志会被误读成"起了多个进程"。

### 【低】S9 观测/指标缺失
- 出问题部分：`proposal.md:38-40` 的"预期"没有量化口径。
- 后果：`server.mjs:62-82` 已经给出每请求 `duration_ms`/`slow` 的落点（`DSH_REVIEW_SLOW_REQUEST_MS`，`server.mjs:18/31`），"首屏变快"完全可以被它度量；但方案没有给出**改动前基线**，结论无法验证（第 5 条验证纪律）。
- 建议：先拿 `docker compose logs web | grep http_request` 里冷启动后前 N 个请求的 `duration_ms` 分布当基线，预热后对比；否则这条"预期"不成立也不可证伪。

### 【低】S10 文档/运维面同步
- 出问题部分：方案没有任何部署面说明。
- 后果：`docs/docker约定.md:6-16` 的"不变量"表（迁移是独立一次性进程 / web 不自己迁移 / 健康检查口径）与新增启动步骤相互关联却未更新；`.env.example:102-104` 的健康口令、`scripts/diagnose.mjs:36` 的 `STALE_MS` 等运维参数也应同步说明预热超时/开关（例如 `DSH_REVIEW_WARMUP_TIMEOUT_MS`、`DSH_REVIEW_WARMUP=off`），保证"线上配置 = 仓库配置"这条不变量不破。
- 建议：把预热写成可关闭、可调超时、可在 `diagnose.mjs` 里观测的三件套。

---

## 三、结论
方案方向（冷启动变快）可以讨论，但**当前形态不可采纳**：失败策略（exit 1）与两个部署形态的重启策略直接冲突，预热对象有三处与代码基线不符且其中 `listPlugins` 是最重的一次查询，还与 `src/instrumentation-node.ts` 既有的"带病启动 + 健康端点报错"设计纪律反向而行。最小可落地改法：**把预热并入 `registerNodeRuntime()`、幂等执行一次、带总超时、失败只记日志与健康 check，绝不 exit**；并且在采纳前先给出冷启动 `duration_ms` 基线，证明收益存在。

---

## 四、L1 直跑留痕
按工作纪律第17条，本次全部为只读判定（read/grep/glob + 认知图 route），产物落点不改仓库、不改外部状态：
「L1 直跑：read/grep/glob 全仓只读检索（src/、deploy/、docker/、scripts/、docs/、package.json）+ cg route — 风险=无（纯读）、频次=本次 8 次、可逆性=完全可逆」
