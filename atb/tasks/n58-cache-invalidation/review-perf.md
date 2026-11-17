# 评审：N-58《插件列表缓存加失效通知》— 性能工程师

评审对象：tasks/n58-cache-invalidation/environment/proposal.md（48 行，全部内容为 §1 缓存 / §2 失效 / §3 跨进程）
本报告所有结论均已对照本仓代码核实；标注“实测”的数值为本次在本机 better-sqlite3 13 上跑出的基准。

## 摘要（前 5 条最重要）

1. 【严重】本方案覆盖的对象不是列表页在跑的查询，缓存命中率≈0 — 依据：src/app/plugins/page.tsx:175 传的是分页/搜索/分类/标签/筛选/排序全参数；service.ts:784-832 的 listPlugins 每次按 `${clause}` 与 LIMIT/OFFSET 现拼 SQL。proposal.md:10-18 缓存的是 `SELECT * FROM plugins` 单变量、无 key — 建议：要么放弃缓存，要么按 query 归一化 key，且先量命中率再存。
2. 【严重】“users_version 做跨进程信号”会触发写放大 — 依据：src/server/platform/db.ts:54 `synchronous = FULL`，proposal.md:36 的 `db.pragma("user_version = ...")` 每次失效都等于一次 WAL commit + fsync（实测 0.906ms/次）。失效通知比省下的查询还贵 — 建议：删除该设计，见第 4 条。
3. 【严重】读信号本身是每次请求一次 pragma 调用 — 依据：proposal.md:40 在 listPlugins 顶部无条件下 `PRAGMA user_version`；实测 0.0180ms/op，而一次 COUNT(*) 裸表仅 0.0104ms — 建议：任何跨进程信号都别放在热读路径上无条件执行。
4. 【严重】端到端算账：本方案对真实页面是负收益 — 依据：实测 COUNT+JOIN 7.6949ms（且这是**最便宜**的一段），proposal.md:10 的缓存根本不命中这 7.7ms，所以“ms→μs”（proposal.md:47）不成立 — 建议：优化点应落在 service.ts:788 的 COUNT 与 4 个相关子查询，而不是缓存裸表。
5. 【严重】进程内缓存 + 60s TTL 在双进程部署下必然给出过期数据 — 依据：docker/compose.yaml:44-108 的 web+worker 两常驻服务共享 .data，deploy/ 另有 systemd unit；proposal.md:36 唯一“跨进程”通路是 user_version，且 proposal.md:32-43 第二版把 invalidate 覆盖了（重复定义，见第 9 条）— 建议：先确认部署形态再谈失效模型。

## 详述

### 1【严重】前提不成立：缓存的目标对象不存在
- 页面查询形状：src/app/plugins/page.tsx:175-186 传 page/page_size/q/cat/tag/hasReviews/assessment/security/sort/dir 共 10 维。
- service.ts:784-830 每次按条件拼 SQL，桩值是 `LIMIT ? OFFSET ?`；proposal.md:13-18 的 listPlugins 签名无参、命中条件只有“60s 未过期”。
- 结论：`SELECT * FROM plugins`（proposal.md:15）既不是页面查询，也无法服务任意筛选组合。真实命中只会发生在**首页无筛选**这一种情形，其余全部穿透。

### 2【严重】缓存的是裸 plugins 表，展示字段全部缺失
- migrations/0001_baseline.sql:87-95 建表时 plugins 只有 id/repository_id/plugin_path/created_at/updated_at 五列；展示字段是 migrations/0005_plugin_directory_info.sql:44-53 才 ALTER 加的，且 repositories 侧的 description/topics/owner_avatar_url 在另一张表。
- service.ts:818-828 实际 SELECT 走 `plugins p JOIN repositories r`，另带 4 个相关子查询（service.ts:137-147 / 153-167 / 170-181 / 184-194）。
- proposal.md:15 的 `SELECT * FROM plugins` 永远拿不到 full_name / topics / public_review_count / assessment_tier_value / security_audit_count / assessed_count，渲染即错误或空。

### 3【严重】写 user_version 在 synchronous=FULL 下是最贵的一类失效通路
- db.ts:53-54：`journal_mode = WAL` + `synchronous = FULL` —— 每次提交都 fsync WAL。
- 实测（本机 better-sqlite3 13.0.3，synchronous=FULL，20000 行）：写 pragma user_version = **0.906ms/次**；只读 pragma = 0.0180ms/次。
- 也就是说，“失效通知”的写成本 ≥ 一次常规 COUNT，而 invalidation 往往紧跟在业务写入之后触发，等于**在每次数据变更后多摊一次 fsync 的尾延迟**。

### 4【严重】跨进程信号：仓库已有等价且零成本的实现
- src/server/platform/db.ts:67-74 是进程级单例，web 与 worker 各自一个进程 —— 双进程共享一个库文件是既定架构（db.ts:66 注释“Web 与 worker 各自一个进程”）。
- **更好实现（明确位置）**：库里已有 `schema_migrations` 表与 `applied_at`（src/server/platform/migrate.ts:23、:95），`MAX(applied_at)` 就是现成的跨进程版本号；如果确实需要信号，应从这里取或**新建一张专用的 cache_epoch 单行表**，而不是借用 `user_version`。
- 为什么要避开 user_version：migrate.ts:256-259 的 selfCheck 虽然只读 schema_migrations，但 `PRAGMA user_version` 是 SQLite 内建的 schema 版本位（与 schema_migrations 表是两套账）。一旦有别处（例如未来的迁移脚本、`PRAGMA user_version = 0` 复位、第三方备份工具）开始使用它，两套账会互相踩，且**症状是“缓存莫名不失效”，排障成本极高**。
- 另：任何“每次请求都读一次信号”的设计，其下限就是 0.018ms/请求；在本页总成本（7.7ms COUNT + N 个子查询）里占比 < 1%，能省但省不了本质。

### 5【严重】端到端算账：真正的开销在 COUNT 与相关子查询，不在“查 plugins”
- 实测（20000 行）：裸表 COUNT(*)=0.0104ms；`COUNT+JOIN repositories`=**7.6949ms**。变慢的原因是 JOIN 让 COUNT 走行迭代而不是覆盖索引。
- service.ts:788 正是 `SELECT COUNT(*) ... FROM plugins p JOIN repositories r ...`，即分页头那 7.7ms 是**每页请求固定**成本，且带 WHERE 时还会更贵。
- proposal.md:47 宣称“列表查询从 ms 降到 μs”：缓存裸表连这 7.7ms 都碰不到，**μs 结论不成立**。
- 更值得做的（按收益排序，均不引入一致性风险）：
  1. 把 service.ts:788 换成覆盖索引 COUNT 或定时物化的计数（这是唯一能吃掉 7.7ms 的地方）；
  2. 检查 service.ts:137-194 四个相关子查询是否有 `reviews(plugin_id, published_revision_id)`、`content_entities(withdrawn_at,hidden_at)` 支撑索引（每行一次子查询 × page_size 行）；
  3. 才是进程内缓存，且必须 key 化 + 有界。

### 6【高】缓存无上限、无 key、无失效钩子的失效面
- proposal.md:10-11 是模块级单变量；proposal.md:25-27 的 invalidate 无参数、整体清空。一旦按 query 扩展（分页/筛选），就变成无限 key 空间 — 必须加 LRU + 上限，否则是内存泄漏。
- 更根本的行为问题：proposal.md:22 “数据变更时主动清缓存”只在走 invalidate() 的代码路径生效。运维直接改库、迁移脚本、worker 进程的写入都**不会**通知 web 进程 —— 60s 内列表页给出过期数据，且无任何日志提示。

### 7【高】§3 代码片段自相矛盾，无法直接落地
- proposal.md:25-28 与 proposal.md:34-37 重复定义 `invalidate`（两次 export 同名函数，后者覆盖前者）。
- proposal.md:41 用到 `cachedVersion`，但 48 行全文**从未定义、从未赋值**；proposal.md:39-43 的 listPlugins 也缺 `cachedAt` 那版的 TTL 分支。
- 结论：§3 不是可评审的设计，是一段残稿。至少应给出完整函数体 + 一次成功/一次失效的时序。

### 8【中】“ms→μs”缺基线数据
- proposal.md:46-48 全是定性结论：没给当前 P50/P99、没给 QPS、没给 N。性能方案没有基线数字就无法验收，建议先补 `/api/plugins` 与 `/plugins` 的实测耗时与调用量，再决定要不要缓存。

## 对性能工程师而言的结论
这是一次**倒退**：为一个并不存在的热点（裸 plugins 表）加缓存，换上一条每次请求都要读、每次失效都要 fsync 的信号通路，并且缓存产物连页面要的字段都不全。真正的 7.7ms 在 service.ts:788 的 JOIN COUNT，本方案完全没碰到。

建议把 N-58 的标题改为“目录页查询成本优化”，范围收窄为：COUNT 物化 + 相关子查询索引核查 + 基线埋点；缓存只作为第三选项，且明确 key 空间与失效边界。
