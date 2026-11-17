# N30「插件列表页分类查询优化」方案 · SRE（运维）评审

> 评审对象：`atb/tasks/n30-n-plus-one/environment/proposal.md`（35 行）
> 评审身份：SRE（运维：根因定位 / 索引与观测 / 变更与回滚 / 容量）
> 事实基线：`src/server/modules/catalog/service.ts`、`src/app/plugins/page.tsx`、`migrations/0010_plugin_categories.sql`、`migrations/0001_baseline.sql`、`migrations/0005_plugin_directory_info.sql`、`src/server/platform/{db,tx}.ts`、`server.mjs`

## 0. 结论

**不建议实施——这份方案优化的是一个不存在的性能问题。**

结论可以一句话说完：**列表页没有 N+1，分类早就是一次批量查询完成的**。方案第 9–31 行描述的"优化前写法"（`SELECT * FROM plugins` + 循环查分类）在本仓库**不存在**；真实的 `listPlugins` 是「1 条聚合 SQL（含 4 个相关子查询）+ 1 条分类批量查询 + 1 条 overrides 批量查询」，而分类这条**本来就用 `IN (...)` 且已经处理了空数组**。见 §1、§2。

如果按方案实施，收益是 **0**（没有可省的查询），风险却是**实打实的**：`plugin_categories` 根本没有 `repository_id` 列（§2），照抄会直接 500；而且方案会**把团队的注意力从真正的瓶颈（4 个逐行相关子查询）上移开**（§3）。

SRE 视角最重要的一句：**方案通篇没有一次测量。** 本仓 `server.mjs` 已经为每个请求打了 `duration_ms`（§4），一次 `EXPLAIN QUERY PLAN` 就能把这件事定死——方案却从"怀疑是 N+1"直接跳到了"改代码"。

---

## 1. 前提不成立：列表页根本不是方案描述的那种写法 —— proposal 第 3–19 行

**① 问题**：方案第 5 行说「按目前的写法，一页 20 个插件要查 21 次库」，第 12–18 行给出"优化前"的 `for (const p of plugins)` 逐行查询。**本仓库没有这段代码。**

**② 后果（核实依据）**：
- 真实的列表入口是 `src/app/plugins/page.tsx:175-194` → `catalog.listPlugins`。其实现（`src/server/modules/catalog/service.ts:784-897`）是：
  - :788 一条 `COUNT(*)` 取总数；
  - :816-832 **一条**主查询，SELECT 里内联 4 个相关子查询（:822-825 引用 `PUBLIC_REVIEW_COUNT_SQL` 等）；
  - :858-861 **一条**分类批量查询（`categoriesByRepository`，:248-269）；
  - :863 **一条** overrides 批量查询。
  **没有任何 per-row 循环查询。**
- 方案里 `db.prepare('SELECT * FROM plugins LIMIT 20')` 这种写法在全仓不存在；`plugins` 表也没有 `repository_id` 之外方案假设的字段语义（见 §2）。
- 更要紧的是：**"21 次 vs 2 次"这个对比本身就是错的**。真实页面的查询数是 4 条，不是 21 条；方案把 4 条"优化"成 3 条（或 4 条，因为分类本来就在里面），收益 ≈ 0。

**③ 建议**：
- 方案的前提段必须改为**实测事实**：贴 `EXPLAIN QUERY PLAN` 输出与一次真实请求的 SQL 条数（可以从 `server.mjs` 的 `duration_ms` 反推，见 §4）。
- 如果目标是"列表页变快"，必须先把**耗时分摊到具体语句**（见 §3），否则无法证明改完之后变快了。

---

## 2. 假设的表结构与本仓不符：`plugin_categories` 没有 `repository_id` —— proposal 第 16、27–29 行

**① 问题**：方案在 `:16` 与 `:27-28` 两处都写 `WHERE repository_id = ?`。**本仓的 `plugin_categories` 没有这一列。**

**② 后果（核实依据）**：
- `migrations/0010_plugin_categories.sql:19-27` 的建表语句是：`slug / label / topics(JSON 数组) / sort_order / is_visible / created_at / updated_at`。**没有 `repository_id`**。
- 分类与插件的关联方式是**通过 GitHub topics 间接匹配**：`plugin_categories.topics`（JSON）∩ `repositories.topics`（JSON），实现在 `catalog/service.ts:248-269`：
  `JOIN json_each(r.topics) rt JOIN plugin_categories c ON c.is_visible = 1 JOIN json_each(c.topics) ct ON ct.value = rt.value`。
- 所以方案的两段 SQL **都无法执行**：`SQLITE_ERROR: no such column: repository_id`。按第 16 行改会 500；按第 27–29 行改同样 500。
- **这条是止血项**：如果实施者只把"优化后"代码贴进生产，列表页会**全线 500**，而这是一个"看起来只是查询优化"的变更——从变更风险来说，这是最糟糕的一类：**低审查强度的改动造成高影响故障**。

**③ 建议**：
- 方案里删掉自造的表结构，改为引用 `0010_plugin_categories.sql:19-27` 的真实列，并把关联语义写成"topics 交集"。
- 顺带补一个真实的**"未分类"判据**：`service.ts:295-302` 用 `NOT EXISTS (... json_each ...)` 表达"没命中任何分类"。若将来真要"优化分类查询"，优化对象是这个表达式，不是 `repository_id = ?`。

---

## 3. 真正的瓶颈没有被识别：4 个**逐行**相关子查询 —— proposal 第 33–35 行

**① 问题**：方案第 34 行断言「查询从 21 次降到 2 次 → 列表页显著变快」。但它**从未指认真正的成本来源**。

**② 后果（核实依据 + 为什么它才是成本）**：
- `catalog/service.ts:137-194` 的 4 个相关子查询（`PUBLIC_REVIEW_COUNT_SQL` / `PUBLIC_ASSESSMENT_TIER_SQL` / `PUBLIC_SECURITY_AUDIT_COUNT_SQL` / `PUBLIC_ASSESSED_COUNT_SQL`）——**每一个都在主查询的 SELECT 列里，对每一行执行一次**（:822-825）。它们是 4 次 join + 聚合，**而且三处复用同一表达式**（SELECT / WHERE / ORDER BY，:314-322、:813）。
- 也就是说：**真实成本 ≈ 页内行数 × 4 次聚合**，而方案要"优化"的分类查询**本来就只有 1 条**。把 4 条查询优化成 3 条，改动的是**常数项**；真正随数据增长的是那 4 个逐行子查询。
- 而且这些子查询**没有可用的覆盖索引**去加速"按插件聚合其公开评测"：`migrations/0001_baseline.sql:159` 只有 `idx_reviews_plugin(plugin_id, id)`，而子查询还需要 join `review_revisions`（:179 的索引是 `(plugin_id, submitted_at, id)`）与 `content_entities`（:147 是 `(content_type, public_updated_at, id)`）。**4 个子查询各自都要走这段 join**，这是全表最贵的一段。
- 结论：方案即使"成功"，**页面也不会有可感知的变快**；而真正的优化方向（把那 4 个子查询聚合成 1 个、或加覆盖索引/物化计数）方案完全没有提。

**③ 建议**：
- 在方案里加一节「成本分解」：用 `EXPLAIN QUERY PLAN` 贴出主查询的子查询节点数，给出"行数 × 4"的成本模型。
- 若要做，优先做**两件事**（按收益排序）：(a) 把 4 个语义相同的子查询合并成 1 个 `LEFT JOIN (SELECT plugin_id, COUNT(*)..., MAX(CASE...) ... GROUP BY plugin_id)` 派生表；(b) 评估 `reviews(plugin_id, published_revision_id)` 上的部分索引，让 4 个子查询共享同一条索引路径。
- **不要把"查询条数"当成优化目标**——SQLite 进程内调用是微秒级，条数从 4 降到 3 不改变 p95；改变 p95 的是**每行聚合的成本与行数**。

---

## 4. 没有测量：本仓已有现成的观测手段，方案一个都没用 —— proposal 第 5、33–35 行

**① 问题**：方案第 5 行「按目前的写法」、第 34 行「显著变快」——**没有任何一次实测**，也没有说明"显著"是多少。

**② 后果**：
- 本仓**已经**具备定位手段：`server.mjs:62-82` 每个请求一行 `http_request`，含 `duration_ms` 与 `slow` 标记（:66-67，阈值 `DSH_REVIEW_SLOW_REQUEST_MS` 默认 1000ms），慢请求自动升 `warn` 落 stderr。**一次部署后看日志就能知道列表页的真实 p50/p95**，方案不用。
- 没有基线就没有"优化成功"的判据：SRE 无法在变更评审上说"这条改动安全"，也无法在变更后判断"要不要回滚"。这是运维最不能接受的状态——**改动不可验证 = 改动不可回滚**。
- 本仓还有一个**隐藏前提**容易被忽略：`better-sqlite3` 是**同步**驱动（`db.ts:6-8` 注释原文「等待会阻塞所属线程」）。所以"查询次数"与"查询耗时"在这里是**直接相加**的，不存在并行掩盖。一条 8ms 的查询与八条 1ms 的查询对 p95 的贡献相同——**方案优化的是计数，不是时间**。

**③ 建议**：
- 方案必须先给出**基线**：`/plugins` 在 N 行/页下的 p50/p95（从 `server.mjs` 的 `duration_ms` 取），以及 `EXPLAIN QUERY PLAN` 的原样输出。
- 给出**目标**与**判据**：例如"p95 从 X ms 降到 Y ms，且 `slow:true` 的比例从 a% 降到 b%"。
- 在 `scripts/diagnose.mjs` 或一个一次性脚本里加"列表页慢查询归因"，让下一次类似问题 5 分钟内能定位，而不是靠猜（这也是本仓"可观测性反哺"的既有做法）。

---

## 5. 空数组保护：方案缺失，但**本仓已经处理过**（这条不是新缺口，是"照抄基线会退步"） —— proposal 第 26–29 行

**① 问题**：方案第 26-29 行 `ids.map(() => '?').join(',')` 在 `ids` 为空时生成 `IN ()` → SQLite 语法错误 → 500。

**② 后果（核实依据）**：
- 这是**方案自身**的缺陷，但它同时暴露了一个更重要的事实：**本仓已经正确地处理过这个问题**。`catalog/service.ts:250` 的 `categoriesByRepository` 第一行就是 `if (repositoryIds.length === 0) return out;`——空数组直接返回空 Map，**根本不发查询**。
- 换句话说：`categoriesByRepository` 的这条路径是"0 条查询"（比方案声称的"2 条"还好），而方案要重写它，**会把一个已经处理好的边界重新变成 500**。在 SRE 眼里这是**功能退步**，不是优化。

**③ 建议**：不要重写 `categoriesByRepository`。如果确实要动，先写失败用例（空页、单行、同仓库多插件）钉住现有行为，再改。

---

## 6. `SELECT *` 与查询变更的运维风险：列漂移会静默扩散 —— proposal 第 13、25–29 行

**① 问题**：方案两段 SQL 都用 `SELECT *`。

**② 后果**：
- `plugins` 与 `plugin_categories` 都会随迁移增加列（`plugin_categories` 在 0011 里被 UPDATE 过，未来还会加）。`SELECT *` 让"表结构"与"代码契约"隐式耦合：加一列不会报错，但会把新列带进结果集、影响后续分组逻辑与响应体积。
- 对运维来说这是**变更期的隐形风险**：一次与本需求无关的迁移，可能让这个查询的返回变大，进而让响应体积/缓存失效。而本仓的既有实现是**显式列出列名**（`service.ts:818-825`），正因为这个原因。
- 另外：`categoriesByRepository` 返回的只是 `{slug, label, sort_order}`（:254），**不返回整行**——`SELECT *` 会把 `topics`（可能几十个大 topic 的 JSON 数组）也搬出来，每个仓库一次。这是**实打实的 I/O 放量**。

**③ 建议**：显式列名；在方案里把"返回哪些列、为什么"写清楚（参照 `service.ts:248-269` 的取舍注释）。

---

## 7. 与真实页面的整体成本：一次渲染串了 4 次读取，方案只谈其中 1 次 —— proposal 全篇

**① 问题**：方案把"列表页"当成"一条查询"，但实际的页面渲染是**串行的多次库读取**。

**② 后果（核实依据）**：
- `src/app/plugins/page.tsx:175-194`：`listPlugins`（:175）→ `listCategories`（:188）→ `featuredCountsForPlugins`（:190）→ 条件分支里**再来一次** `listPlugins`（:194，仅在 `result.total === 0` 时）。而且这些都是 `await` 串行。
- 加上 `listPlugins` 内部的 4 条 SQL，一次"筛选后为空"的页面要跑 **7 条查询**。方案优化的那条分类查询只占 1 条，而且它本来就不是瓶颈。
- 又因为同步驱动（`db.ts:6-8`），这些查询**完全串行**，没有并行掩盖。所以"减少条数"只有在减少**最贵那条**时才有意义。

**③ 建议**：方案的"成本表"应该按页面维度列全（每条语句 + 行数 + 耗时），而不是只盯一个点。真正的低成本高收益项可能是：把 `:194` 的第二次 `listPlugins` 换成轻量计数查询（它只为拿 `total`，却跑了一整套聚合子查询）。

---

## 8. 变更与回滚：这是"优化"，必须可量化回退 —— proposal 全篇

**① 问题**：方案没有写回滚判据、没有写兼容面。

**② 后果**：
- 本仓部署是「`migrate` 一次性进程 → web/worker 等它成功」（`docker/compose.yaml:44-48,78-83`），`web` 是 `restart: unless-stopped`（:68）而 systemd 侧是 `Restart=always`（`deploy/dsh-review-web.service:20-21`）。**如果这次的"优化"引入 500（如 §2 的列不存在、§5 的空数组），表现是"页面对所有人 500"**，而重启救不了——只有回滚代码能救。
- 方案没有写"出问题时怎么判断是这次改的"，也没有写"回滚后如何确认已恢复"。

**③ 建议**：在方案里补三条：
1. **上线前后对比**：同一组 URL 的 `duration_ms` p50/p95；
2. **错误率**：`server.mjs:67` 的 `status >= 500` 行数（部署窗口 ±30 分钟）；
3. **回滚判据**：出现任一 500 或 p95 无改善 → 立即 `git revert` + 重建，不做"再调一版试试"。

---

## 9. 一句话给作者

这份方案的三段代码各自对应一个可核实的事实：**"优化前"的 N+1 不存在**（`service.ts:784-897` 全是批量）、**"优化后"的列不存在**（`0010_plugin_categories.sql:19-27` 没有 `repository_id`）、**真正的成本没被看见**（`service.ts:137-194` 的 4 个逐行子查询）。**要做性能优化，第一步永远是 `EXPLAIN QUERY PLAN` + 一次真实 `duration_ms` 基线——本仓的 `server.mjs` 已经把这条基线打印出来了，方案只要去读它。**

## 10. 附：建议的方案骨架（供改写参考）

| 步骤 | 产出 | 验收 |
|---|---|---|
| 1. 测量 | `EXPLAIN QUERY PLAN` + 页面 `duration_ms` p50/p95 | 基线数字落在方案里 |
| 2. 归因 | 按语句列出耗时与行数（含 4 个相关子查询） | 说出"最贵的是哪一条" |
| 3. 改造 | 合并 4 个子查询为 1 个派生表 / 加覆盖索引 / 去掉 `:194` 的重复查询 | 改动限定在 `catalog/service.ts` 与 `migrations/` |
| 4. 对比 | 同口径重测 | p95 改善 ≥ 方案目标，且 500 数为 0 |
| 5. 回滚 | 判据与步骤 | 出现 500 或无改善即回滚 |
