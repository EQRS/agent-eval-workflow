# n30-n-plus-one · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0010
> 模型 deepseek-chat · 5188 字 · 21 次工具调用

---

已核实。以下是我以 SRE 身份进行的评审。

---

# SRE 评审意见：插件列表页分类查询优化

**先说结论：方案里那段"优化前"的代码无法执行**（它查询的列在 schema 中不存在）。因此，它并非"从 21 次降到 2 次"的性能问题，而是对现行设计的误解。如果按方案实施，列表页会直接返回 500 错误。建议方案整体重写。

---

## 问题 1（阻断级）：`plugin_categories` 根本没有 `repository_id` 列

**① 哪部分有问题**
方案第 1 节（"优化前"）和第 2 节（"优化后"）都写了：

```ts
'SELECT * FROM plugin_categories WHERE repository_id = ?'
```

`proposal.md:14-16`、`proposal.md:28-30`。

**② 后果**
实际 DDL（`migrations/0010_plugin_categories.sql:23-31`）中，`plugin_categories` 只有 6 列：`slug / label / topics / sort_order / is_visible / created_at / updated_at`。

- **没有** `repository_id`、**没有** `plugin_id`。
- 它是**分类字典表**（10 行种子数据，`0010:38-47`），并非"插件与仓库的关系表"。

所以那两条 SQL 都会在 `prepare()` 阶段失败，报出 `no such column: repository_id`。以 `better-sqlite3` 的同步驱动，这个错误会同步抛出，导致请求直接返回 500 错误，错误路径是 `db.prepare` → `better-sqlite3` → `openDatabase`（`src/server/platform/db.ts:45-71`）。

进一步说，`plugin_categories` 只有 10 行，而仓库数量是数百量级，因此"按 `repository_id` 查分类"这件事**在语义上不存在**。

**③ 建议**
方案必须基于真实的设计重写。真实的关联链路是**通过 JSON topics 做双向匹配**，见 `src/server/modules/catalog/service.ts:243-262`：

```sql
JOIN json_each(r.topics) rt
JOIN plugin_categories c ON c.is_visible = 1
JOIN json_each(c.topics) ct ON ct.value = rt.value
```

也就是说：`repositories.topics`（仓库的 GitHub topics）与 `plugin_categories.topics`（该分类认领的 topic 白名单）取交集。请删除"`plugin_categories` 有 `repository_id`"这个前提，不要再围绕它改 SQL。

---

## 问题 2（阻断级）：N+1 已经修过了，方案是在重做已存在的正确实现

**① 哪部分有问题**
方案称"目前的写法一页 20 个插件要查 21 次库"（`proposal.md:4-5`），并把逐行查询当作现状。

**② 后果**
现状**并不是**逐行查询。`service.ts:243-275` 的 `categoriesByRepository()` 注释明确写着：

> `批量取「仓库 → 命中的分类展示名」映射（**一次查询**，避免 N+1）`

实现也是单条 `WHERE r.id IN (...)` 批量查（`service.ts:255-262`）。而且它做得比方案更对：**按 `repository_id` 去重后查一次**，因为按 `plugin_categories` 的语义"同一仓库的 11 个 packages 会得到同一组分类"（`service.ts:248-251`）。

如果按方案把它"改成" `WHERE repository_id IN (...)`，等于用一条不存在的列去替换一个已经跑通的单查询，这是在**引入回归**，而不是修复性能问题。如果 N+1 确实在线上观测到了，那它在别处（比如详情页、worker 的 `discovery.ts`），需要重新定位。

**③ 建议**
方案的第一步应当是**复现现状**：贴出真实的 `EXPLAIN QUERY PLAN` 和慢查询采样，证明 N+1 确实存在、以及出在哪个函数。没有这份证据，不应改动 `categoriesByRepository()`。

---

## 问题 3（高风险）：`IN (${ids.map(...)})` 这个拼接是维护隐患，且不解决真正的热点

**① 哪部分有问题**
`proposal.md:28-31` 用模板字符串拼占位符：

```ts
`WHERE repository_id IN (${ids.map(() => '?').join(',')})`
```

**② 后果**
- 值本身是绑定的（`...ids`），所以**不是 SQL 注入**，这点不含糊。
- 但**语句形状随 `LIMIT` 变化**，每变一次页大小就是一条新的 prepared statement，`better-sqlite3` 的语句缓存会不断命中新键、旧条目老化，`prepare` 开销反复发生。
- 更要紧的是：分类查询的真实成本在 `json_each` 展开，与 `IN` 列表长度无关。`repositories` 表上现有索引只有 `idx_repositories_availability (availability_state, id)`（`0001_baseline.sql:84`），**没有针对 `topics` 的任何索引**，而 `json_each(r.topics)` 是全表每次都要展开的。方案宣称"显著变快"，但真正慢的那一步一字未提。

`plugins` 侧同样只有 `idx_plugins_updated (updated_at, id)`（`0001:95`），`0005` 又加了 `idx_plugins_updated_id (updated_at DESC, id ASC)`（`0005_plugin_directory_info.sql:57`）。**`plugins.repository_id` 上没有任何索引**，而 `0001:93` 的 `UNIQUE (repository_id, plugin_path)` 恰好能为按 `repository_id` 前缀查询服务，这一点方案没有利用也没有说明。

**③ 建议**
- 保持现有单查询形态；如果确实要调优，目标是 `json_each`。
- 若 `repositories.topics` 匹配成为热点，把"仓库 → 分类"的命中结果**物化**成一张派生表（`repository_categories(repository_id, slug)`，带索引），在 topics 写入时同步刷新。这样列表页就是普通索引扫描，才是真正的"2 次查询"。
- 这需要新迁移。按 `0011_plugin_categories_expand.sql:15` 的既有规范（"迁移一旦应用就不再修改"），必须**新建 `0043_*`**，不得改 `0010/0011`。

---

## 问题 4（中）：读路径的并发与琐碎查询会放大，但方案没有任何运行时可观测项

**① 哪部分有问题**
方案只有一句"列表页显著变快"（`proposal.md:34`），没有延迟目标、没有查询次数验证方式、没有回滚条件。

**② 后果**
- 连接是**进程级单例**，业务与列表页共用（`db.ts:73-81`）。`better-sqlite3` 同步阻塞所属线程（`db.ts:6-7` 明确写了这条纪律）。列表页查询变慢会直接顶住整个 web 进程的事件循环，不只是"这个页面慢"。
- 写连接用 `synchronous = FULL`（`db.ts:60`），WAL 下写会与读争锁，`busy_timeout` 5000ms（`db.ts:64`）。列表页是高频只读，一旦它变成长查询，会把写侧挤到超时。

**③ 建议**
- 方案中补上验收口径：`EXPLAIN QUERY PLAN` 前后对比、p95 延迟基线、`-wal` 文件增长观察窗口。
- 明确"只读走 `withRead`（`tx.ts:52-54`）、不进写事务"。
- 给出回滚方式：这是纯代码改动，可以按开关回退；若引入物化表，必须有"删表并回退读旧路径"的迁移说明。

---

## 问题 5（中）：`LIMIT 20` 无 `ORDER BY`，翻页会是随机结果

**① 哪部分有问题**
两段代码都是 `SELECT * FROM plugins LIMIT 20`（`proposal.md:14`、`proposal.md:25`），**没有 `ORDER BY`**。

**② 后果**
SQLite 对无 `ORDER BY` 的 `LIMIT` 不保证稳定顺序。后果是：同一用户刷新页面，**分类徽章会跳到不同插件上**；而且这个问题无法复现，会被当成"分类匹配错了"报给运维。

现有的目录浏览实现明确用 `(updated_at DESC, id ASC)` 并"带 id 兜底保证不重不漏"（`0005_plugin_directory_info.sql:56`、对应索引 `0005:57`）。方案却绕开了这套既有口径。

**③ 建议**
用既有浏览序 `ORDER BY p.updated_at DESC, p.id ASC`，直接吃到 `idx_plugins_updated_id`。同时**不要写 `SELECT *`**，按 `0005` 的分工只取需要的列（`repositories.*` 是仓库级、`plugins.*` 是插件级，见 `0005:13-16`）。

---

## 问题 6（中）：方案会掩盖 `__none__` 的两种语义，影响运维排查

**① 哪部分有问题**
方案把分类简化为"一个插件一组分类"，完全没有提到"未分类"。

**② 后果**
代码里 `UNCATEGORIZED_CATEGORY = '__none__'` 和 `UNCLASSIFIED_TAG = '__none__'` 是**两个不同的运维含义**（`service.ts:59-76`）：

- 前者 = 有 topics 但**没一个被分类认领** → 运营待办；
- 后者 = **根本没有 topics** → 抓取/探针故障。

`service.ts:70-73` 特别注明"两者都不该被悄悄合并"。如果这次改动把分类变成"逐插件挂表"，`__none__` 的两种语义会混成一种，**抓取故障就不再能被运营指标发现**。

**③ 建议**
方案必须写明：不改动 `__none__` 的两种分支语义（`service.ts:264-284` 的 `NOT EXISTS` / `EXISTS` 两条路径），并在改动后跑 `tests/catalog-browse.test.ts` 回归。

---

## 需要方案作者补充的信息

1. **N+1 的实证**：来自哪条 URL、哪次采样、`EXPLAIN QUERY PLAN` 输出。
2. **"21 次"是怎么数出来的**：现有代码是 `1 + 1` 次（`service.ts:243-262`），与 21 相差太远。
3. 是否已知 `plugin_categories` 是字典表。这是判断方案是否在对的方向上的**唯一前提**。

**我的建议是：打回重写。** 在真实的设计下（`json_each` 交集），这个改动**没有可做的东西**，除非目标转为"物化 `repository_categories` 派生表"，那是另一件事，需要独立迁移与上线计划。