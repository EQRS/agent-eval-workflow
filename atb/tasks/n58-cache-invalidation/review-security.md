# 评审：N-58《插件列表缓存加失效通知》— 安全工程师

评审对象：tasks/n58-cache-invalidation/environment/proposal.md（48 行：§1 缓存 / §2 失效 / §3 跨进程通知）
评审视角：安全工程师（仅安全面：内容可见性治理失效、安全信号持久化明文化、拒绝服务面、审计与取证完整性、信号被劫持）
结论一句话：这份方案的本体是"性能优化"，但它把**内容下线通道**和**一个全局持久的明文信号槽**绑进了热读路径，安全上的净收益为负。

## 摘要（前 5 条最重要）

1. 【严重】方案把"60 秒缓存"直接架在**公开可见性判据**上，管理员隐藏/作者撤回后，目录仍会继续展示该评测的计数与徽章 60 秒 —— 这是方案自造的、有界但确定的"下线未生效"窗口，而本仓可见性判据是"三个条件同时满足"的硬口径（src/server/modules/content/index.ts:153-166 的 publiclyVisible = published && !withdrawn && !hidden；src/server/modules/content/index.ts:255-285 hide() 只改 content_entities + 写审计），listPlugins 的四段公开子查询（src/server/modules/catalog/service.ts:137-194 的 PUBLIC_REVIEW_COUNT_SQL / PUBLIC_ASSESSMENT_TIER_SQL / PUBLIC_SECURITY_AUDIT_COUNT_SQL / PUBLIC_ASSESSED_COUNT_SQL，全部带 `ce.withdrawn_at IS NULL AND ce.hidden_at IS NULL`）恰好**把 content_entities 的治理状态当判据**；缓存不覆盖该表，且 hide() 里没有任何 invalidate 调用点 —— 建议：任何缓存的分层里，**可见性/治理态一律不进缓存**（或缓存键里带 `visibility_control_version`，src/server/modules/content/index.ts:164/276 已维护该版本号），并显式规定"隐藏/撤回/封禁路径必须先失效后返回 200"，写进附录的触发点清单。

2. 【严重】§3 把缓存失效信号写进 `PRAGMA user_version`，是以**明文、全局、持久**的方式把内部控制面状态落盘 —— 依据：现状是 `user_version` 全仓零引用（grep 无命中，仅有 tasks/n58 自己的文本），本仓 schema 版本走 `schema_migrations` 表（src/server/platform/migrate.ts:21-28、src/server/platform/migrate.ts:255-260，健康面 src/server/platform/health.ts:142 的 schema_version 也取自该表），所以这个槽位在本仓是**免费的**；但 `Date.now()`（~1.7e12）与 `journal_mode = WAL`（src/server/platform/db.ts:53）共同意味着：**每次失效的毫秒级时间戳都明文落在库文件头部，并可能残留在 WAL 帧里**；而备份是整库热备（scripts/backup.mjs:6-11 用 `db.backup()` 全库复制，deploy/dsh-review-backup.service:26 每日跑，--keep 14）与 `DSH_REVIEW_BACKUPS_DIR` bind mount（docker/compose.yaml:67）—— 信号因此**进入备份、进入任何拷库文件的人（含原运维）手上**。这正是本端安全评审第 9 条的形态：含内部/敏感内容只进私有面、不外溢 —— 建议：**删除 §3**；若确需跨进程信号，改用库内专用表（如 `cache_epoch(key, version)`，与 `schema_migrations` 同级），或直接读业务提供的事实表（见第 3 条），永远不要占用 SQLite 保留给 schema 管理的元数据槽；若坚持保留**任何**时间戳信号，方案须写明"该值不得进入对外制品/日志/错误页"，并在备份出口做白名单。

3. 【高】跨进程失效在**真实部署形态**下本来就是另一个进程写的，方案却要求业务代码逐个手工调用 —— 依据：docker/compose.yaml:44-108 有 web 与 worker 两个常驻服务共享同一 `.data`（compose.yaml:65、compose.yaml:92），deploy/ 另有 systemd unit；而 worker 的 `plugin_discovery` 处理器（src/worker/handlers.ts:286-303）走`acceptPluginSubmissionBySystem`（src/server/modules/catalog/discovery.ts:270）**在 worker 进程里写 plugins/repositories**，web 侧`POST /api/plugins` 才写（src/app/api/plugins/route.ts:80）。内容侧同理：管理员的隐藏由 web 写（src/server/modules/content/index.ts:266-279），作者撤回、举报处理、批处理导入也各有写点。§2 的 `invalidate()` 只有一个调用点集合的定义，**没有调用点的清单**，方案也没说明"哪些写路径在别的进程" —— 建议：把失效触发点按进程列成表（web 用户收录 / web 管理员隐藏 / worker 自动发现 / 迁移脚本 / 运维 SQL），并明确"任何一次漏调 = 全站最长 60s 展示过期数据"；更稳的写法是**读侧只依赖业务事实**（例如 listPlugins 结果里已有 `r.last_checked_at`/`p.updated_at`，见 src/server/modules/catalog/service.ts:818-821），不要依赖"写入者记得叫一声"。

4. 【高】§3 的读信号是**无界无退避的库访问**，与 §1 的"缓存只为降负载"自相矛盾，且在 busy 时会被放大成延迟放大面 —— 依据：proposal.md:40 每个请求在 listPlugins 顶部无条件 `PRAGMA user_version`，而 `.data` 的频控表也在这条连接上（src/server/platform/rate-limit.ts:37-43 用同一 SQLite 库即 `rate_limiter_flexible`），写侧 `synchronous = FULL`（src/server/platform/db.ts:54）每次提交都 fsync；`busy_timeout = 5000`（src/server/platform/db.ts:56）意味着**信号读在写热点上最多可等待 5 秒**，而这条路径是公开匿名接口（src/app/api/plugins/route.ts:24-43 无鉴权）—— 建议：任何跨进程信号都必须至少是"有 TTL 的惰性读 + 单航班"，不能放在每个请求的同步前置；公开面读路径**不得**因缓存层引入新的阻塞点。

5. 【中】失效信号可被**任何能写库的进程**推进（迁移器、运维）→ 不受控失效/抖动，且方向是"更贵"而不是"更安全" —— 依据：`PRAGMA user_version` 不需要业务表、不需要审计，写入即生效；本仓迁移器用独立连接的 `openDatabase()`（src/server/platform/migrate.ts:76，迁移容器 `migrate` 在 compose.yaml:42 结束后退出），任何一次 `sqlite3` 手工操作都能改写该值。一个失效 flapping 攻击/误操作即可让缓存命中率归零（退化回方案本要消除的负载），且**不留痕**（不像业务写会进 operation_records）—— 建议：信号源只能来自业务写路径（有审计），不接受任何非业务来源的版本推进；如保留专用信号表，加 `CHECK`/`updated_by` 并纳入 `selfCheck`（src/server/platform/migrate.ts:239-273 已有 required tables + foreign_key_check 的体检位）。

## 详细发现

### 一、前提核实（安全角度的"对象是否存在"）

| 方案假设 | 仓里的事实 | 依据 |
| --- | --- | --- |
| 存在一个"插件列表查询"值得缓存 | 存在，但形状是**按 10 维条件现拼 SQL** | src/app/plugins/page.tsx:175-186；src/server/modules/catalog/service.ts:784-832 |
| 缓存对象 = `SELECT * FROM plugins` | 页面要的是 plugins ⋈ repositories ⋈ content_entities ⋈ reviews 的聚合 | src/server/modules/catalog/service.ts:816-855（四段公开子查询、JOIN repositories） |
| `plugins` 表有页面需要的列 | 基线只有 5 列，展示字段是迁移 0005 之后 ALTER 加的 | migrations/0001_baseline.sql:87-95（其它列见 0005） |
| `user_version` 可以随便用 | 不用**它，但原因是本仓换成了 schema_migrations**；该 pragma 属于 SQLite 文件头，是**全局持久状态** | src/server/platform/migrate.ts:21-28 |
| invalidate() 能覆盖所有写路径 | 至少 4 类写者分布在 3 种进程（web 用户 / web 管理 / worker / 迁移） | src/app/api/plugins/route.ts:80；src/server/modules/catalog/discovery.ts:270（经 src/worker/handlers.ts:303 调用）；src/server/modules/content/index.ts:266-279；src/server/platform/migrate.ts:88-123 |

结论：**"缓存什么"的假设与仓库对不上（这是 L4 层的根因），"怎么失效"的假设与部署形态对不上。** 安全上真正要守的是下面三条不变量，方案一条都没写。

### 二、安全不变量 1：下线必须即时（当前方案违背）

- 判据链：公开可见 = 有已发布修订 ∧ `withdrawn_at IS NULL` ∧ `hidden_at IS NULL`（src/server/modules/content/index.ts:153-166；SQL 同口径见 src/server/modules/catalog/service.ts:137-194）。
- 隐藏是**管理员强制动作**（src/server/modules/content/index.ts:255-285，`requireAdmin` + CAS 版本 + 审计），撤回是**作者主动动作**（同文件:188 起）。两者都属于"必须在秒级对外生效"的治理动作，本仓甚至为此加了 `visibility_control_version` 兜底。
- §1 的 60s 窗口 + §2 的"等人调用 invalidate()"= 把这两个动作的生效时间变成 **最多 60s**。方案 §预期 里写的是"列表查询从 ms 降到 μs / 跨进程一致"，**完全没有提可见性延迟**——这不是性能问题，是治理失效被当作零成本。
- 加重项：`listPlugins` 的公开计数同时驱动"按评测数/档位/安全审计筛选"（src/server/modules/catalog/service.ts:314-322），所以过期数据不只影响展示，还影响**筛选结果集**——被隐藏的评测仍能被"有安全审计"筛出来。
- 建议（最小改动版，若坚持缓存）：缓存键 = `(query, max(p.updated_at), max(ce.visibility_control_version), 全局 cache_epoch)`；或干脆**只缓存不依赖治理态的部分**（仓库元数据），可见性/计数永远现算。

### 三、安全不变量 2：内部信号不得落进持久化/备份面（当前方案违背）

- `PRAGMA user_version` 是 SQLite 数据库文件头里的 4 字节有符号整数——**它是文件的一部分**，不是内存状态。方案写的是 `Date.now()`（约 1.7e12，超出 32 位，SQLite 会按大整数存/回读，但这不影响结论）：无论存成什么，都是一条"系统内部节拍"落在文件里。
- 本仓把整库当"完整恢复点"：`scripts/backup.mjs` 用 `db.backup()`（scripts/backup.mjs:6-11）整库热备并写 manifest（scripts/backup.mjs:11-13），deploy/dsh-review-backup.service:26 每天跑、保留 14 份，落点是宿主 bind mount（docker/compose.yaml:67）。这条信号会随每份备份外流，且**无法用业务语义解释**（干净库里出现 `user_version=1794xxxxxxxx` 会被当成异常/攻击痕迹，干扰取证）。
- 与内容政策/隐私的关系：本条不属"个人隐私"（不是自然人数据），属**内部状态外溢**；按照 N-17 一带的处理口径（`_work/评审报告-n17数据泄漏-安全.md` 的思路：能删就删、不能删就最小化），建议删。
- 建议：用 `schema_migrations` 同一层的**专用表**（带 `CHECK`、带 `updated_by`、纳入 selfCheck 的 REQUIRED_TABLES 见 src/server/platform/migrate.ts:142-237），或者索性从**业务事实**取版本（见不变量 3）。

### 四、安全不变量 3：失效源必须是业务写，且可审计（当前方案违背）

- 方案给出的通路 `db.pragma(`user_version = 1794888715272`)`（proposal.md:36）不需要任何权限校验、不需要任何表、不留审计。也就是说：**任何能打开该写连接或直接改文件的进程都能"触发全站缓存失效"**，且事后 `operation_records` 里查不到（对比：本仓所有治理写都要求 `recordOperationInTx`，src/server/modules/content/index.ts:290-294）。
- 影响不是"越权读数据"，而是**可用性面**：持续推进版本 = 命中率归零 = 退回原负载，且无从归因（运维排查时看到的是"数据库好像被谁动过"）。
- 建议：版本推进点必须与业务事务同源。最好的形态是**复用已有事实**：`p.updated_at`（src/server/modules/catalog/service.ts:818）、`r.last_checked_at`（同:820）、`content_entities.visibility_control_version`（src/server/modules/content/index.ts:164）。三者已经覆盖方案想表达的"数据变了"，用它们做版本不需要新写任何东西。

### 五、拒绝服务 / 放大面

1. **公开查询无鉴权**：`GET /api/plugins` 的 GET 分支不经过 `currentContext()`、不经过频控（src/app/api/plugins/route.ts:24-46；对比同文件 POST 的 :66-73 才限流）。因此热读路径上**任何新增的同步阻塞**都是匿名可达的攻击面。§3 的每请求 pragma 读正好落在这里。
2. **缓存的租户/参数污染风险（未来形态）**：§1 是"单变量整表缓存"，暂时没有 key 空间问题；但本页查询已在 10 维上分叉（src/app/plugins/page.tsx:175-186），一旦改成"按参数缓存"（L28-L30 的判据也在提这点），就会重演经典问题——**key 未归一化导致他人筛选结果串到当前请求**。安全上应预先立规矩：key 必须由服务端解析后的条件对象规范化生成（不能直接拿 URL 串），且携带 user/role 维度（虽然目录是公开的，但 `/me/drafts/**/choose-plugin` 是登录态页面，src/app/me/drafts/[draftId]/choose-plugin/page.tsx:68 也用同一个 listPlugins）。
3. **缓存对象被上层按引用共享**：`listPlugins` 返回的是对象数组；如果做成模块级缓存并复用同一份对象，调用方后续对结果做 `push/sort`（例如分页合并、排序）**会污染缓存**。这不是"读不到"，而是"读到了被别的请求改过的数组"——属于跨请求数据污染。建议：缓存只到"数据库行"层，返回前每次深拷贝，或使用不可变返回类型。当前 §1 直接 `return cache`（proposal.md:14）。
4. **缓存无上限、无 key 管理**（proposal.md:10-18）：就是单变量，安全牵连很小，但一旦按 key 改就是**内存放大**面——匿名可构造无穷 `q` 组合（src/app/api/plugins/search/route.ts:20-21 只 `slice(0,100)` 不做词表限制），逐个 key 塞满进程内存 → 单实例 OOM。建议：明确"缓存最多 N 条 + LRU"写进方案。

### 六、审计与取证

- 方案没有涉及 `operation_records`。安全侧的要求是：**任何影响对外可见性的变更都必须可在审计表里查到**（现状已满足：hide 写 `content.hide`，src/server/modules/content/index.ts:290-294；撤回同理；收录在 `catalog.accept_plugin_submission`，src/server/modules/catalog/service.ts:50）。缓存**不得**让"审计已写、对外仍可见"成为常态——那会让审计记录与线上事实矛盾，事后追责时无法用审计判断"下线是何时生效的"。
- 反向要求：缓存层**不得**写入 `operation_records`（等于把每个请求变成审计行，污染审计表并放大 `synchronous=FULL` 的写成本）。方案里没有这个风险，但要在落地时明确禁止。

### 七、与既有实现的对照（"已有更好实现"在哪）

1. **缓存形态的正确参照**：`src/packages/common/autosave.ts:23-30` 的客户端偏好缓存是**模块级 + 显式失效 + 失败回落默认值**——它的边界被写在注释里（"设置页改完会调 `invalidateAutosaveCache()`"）。提案 §1/§2 缺的正是这两句边界，"失效点清单"和"读失败怎么办"。§1 现在一旦 `db.prepare` 抛错（库被锁、迁移中），**每次请求都会重试并继续抛**，没有不变量可言。
2. **schema 版本的正确载体**：`schema_migrations` + `selfCheck()`（src/server/platform/migrate.ts:239-273）与 `/api/health` 的 `schema_version`（src/server/platform/health.ts:142，src/app/api/health/route.ts:21 显式 `force-dynamic` + `no-store`）。若真要一个"全局版本号"，应当照这个形态做**受检查、可观测**的表，而不是借 SQLite 保留槽。
3. **可见性判据的唯一权威**：`resolveVisibility`（src/server/modules/content/index.ts:171-176）。任何缓存都应当以它为"最终判据"，而不是复制一份带缓存的计数。

## 落地必须补进方案的三句话（可直接抄）

1. 缓存**不得**包含任何由 `content_entities.withdrawn_at/hidden_at` 派生的字段；若包含，失效必须在同一事务提交前完成，不可延迟。
2. 禁止使用 `PRAGMA user_version`（及其他 SQLite 保留槽）做应用信号；应用信号只允许出现在 `schema_migrations` 同级的专用表里，或直接复用业务时间戳/版本列。
3. 任何缓存层上线前，必须给出"失效触发点 × 进程"矩阵与"漏掉触发点时的最大暴露时长"，并声明该时长是**对外可见性承诺**的一部分（而不是实现细节）。
