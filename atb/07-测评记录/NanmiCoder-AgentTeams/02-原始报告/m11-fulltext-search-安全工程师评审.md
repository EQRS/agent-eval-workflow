# m11-fulltext-search 安全工程师评审

> 任务 t10 · 成员 e3-investigator · 状态 completed
> 日期：2026-11-17

---

# m11-fulltext-search 安全工程师评审

评审对象：atb/tasks/m11-fulltext-search/environment/proposal.md（48 行）
实测方式：用项目自带 createTestDb 建真实 FTS5 虚表验证注入与中文分词（临时脚本已删除，仓库零改动）。

## 最重要 5 条

### S1【阻塞·可实测的注入】proposal:39 的 SQL 字符串拼接可被 FTS 语法逃逸
原文：const sql = SELECT plugin_id FROM plugin_search WHERE plugin_search MATCH '${q}'（直接插值）
**实测结论（与直觉不同，必须说清）**：
- 天真 payload `x' OR '1'='1` → 抛 `unable to use function MATCH in the requested context`（**没打通**）；
- 但 `x' OR t MATCH '别的` → **成功执行，返回了不匹配的行**（实测返回 p2）；
- `zzz' OR t MATCH '记忆插件' OR t MATCH 'x` → **绕开用户关键词，直接命中目标行**（实测返回 p1）。
即：攻击者可以**用自己的 FTS 表达式覆盖用户输入**，枚举/探测索引内容。
更糟的是 q 未截断（proposal:35 直接把 q 丢给查询），且 `AND`/`(`/`NEAR(`/`*` 等任意输入都会让
SQLite 抛 fts5 语法错误 → 走 errorResponse 折成 500（platform/errors-http.ts:42）。
② 后果：越权读 + 可被打成 500 的错误面。
③ 建议：**必须**改成参数绑定（conn.prepare('... MATCH ?').all(phrase)）+ 双引号转义。
本仓已有现成正确实现可直接照抄：src/server/modules/search/index.ts:55-58 的 toFtsPhrase
（`return "\"" + input.replace(/"/g,'""') + "\""`）与 :161 的 `WHERE review_search MATCH ?`；
q 还要照 :137 的 `.trim().slice(0,100)` 截断。

### S2【高】缺 tokenize='trigram'，proposal:47「支持中文搜索」不成立
proposal:12-16 的 CREATE VIRTUAL TABLE 没有 tokenize 参数 → 默认 unicode61。
**实测**（索引「记忆插件 / 用于记忆管理的插件」）：
| 查询 | 默认 unicode61 | trigram |
| 记忆插件 | 1 | 1 |
| 记忆（2字） | **0** | 0 |
| 插件（2字） | **0** | 0 |
② 后果：中文用户搜「记忆」「插件」这种常见词**一条都搜不到**，与 proposal:47 的预期相反。
③ 建议：照 migrations/0014_review_search.sql:5-9 的既有取舍——用 tokenize='trigram'，
并对 **1-2 字查询走 LIKE 回退**（0014:8 明确写了 trigram 只对 ≥3 字符生效，
回退实现见 search/index.ts:156-170 的 `if ([...q].length >= 3)` 分支）。
另注意 0014:6-7 的注释：unicode61 把中文整段当一个词，连「记忆插件」搜「记忆」都不行。

### S3【高】索引用 plugins.display_name/description 原值，与站点实际展示不一致（安全相关的信息泄漏面）
proposal:28-29 写 plugin.name / plugin.description。
**代码事实**：plugins.display_name/description 是**同步来的原值**，站内展示时优先用
plugin_field_overrides 的修正值（migrations/0015_plugin_corrections.sql:11-14 明确说明；
读取见 corrections/index.ts:90-107 overridesFor）。
② 后果两个方向都错：
  (a) 管理员已修正的展示名**搜不到**（用户按页面上看到的名字搜，结果为空）；
  (b) 已被覆盖掉的旧值**仍能搜到**——若旧值是错误/误导性内容，等于让它继续可被检索到。
③ 建议：索引时取 COALESCE(override.value, plugins.display_name)，并在采纳/撤销纠错时
同事务 reindex（参考 reindexReviewInTx 与发布流程同事务的做法，search/index.ts:81-87）。

### S4【高】upsertSearchIndex 不是事务，DELETE + INSERT 之间存在窗口，且可产生索引漂移
proposal:23-31 两次独立 await（DELETE 一次、INSERT 一次），无 withTransaction。
**代码事实**：本仓对索引用**同事务**保证（search/index.ts:82-86 注释：“索引与公开修订指针
必须一起成功或一起回滚，否则会出现搜得到但打不开 / 发布了却搜不到”）。
② 后果：(a) 两次调用之间崩溃 → 该插件**永久从索引消失**（DELETE 成功、INSERT 未执行），
用户搜不到但插件存在；(b) 并发 upsert 同一 plugin 可交错出重复行或丢行；
(c) proposal:20「插件新增或更新时同步写入」没有挂在任何既有写路径上，
实际会有大量插件从未建索引（需回填脚本，参考 search/index.ts:120-132 rebuildIndex）。
③ 建议：包进 withTransaction，并挂到插件写入/同步的具体调用点；另加 rebuild 脚本 + 定时对账。

### S5【中】新表未登记 REQUIRED_TABLES，自检会判定缺表
proposal:12 建 plugin_search，但方案没提登记。
**代码事实**：src/server/platform/migrate.ts:175-176 显式登记了 review_search 并注明
「FTS5 虚表；sqlite_master 里同样是 table 类型」。
② 后果：迁移后自检/健康检查会报缺表，部署流程可能直接失败。
③ 建议：在 migrate.ts 的必需表清单里加 plugin_search（挨着 review_search）。

## 完整清单（其余）

### S6【中】查询接口与既有 GET /api/plugins/search 重名且职责冲突
proposal:35 定义 GET /api/plugins/search?q=。
**代码事实**：该路由**已经存在**（src/app/api/plugins/search/route.ts:16-40），
现在走 catalog.listPlugins（:24），已有 q 截断到 100 字（:20）、limit 1..20 钳制（:21）、
参数化 LIKE（catalog/service.ts:276-283 用 escapeLikePattern + ESCAPE '\\'）。
② 后果：按 proposal 实现会**覆盖**一个正在被精选编辑页使用的接口，且新实现安全性低于旧实现。
③ 建议：不新建同名路由；若要做全文检索，作为新参数或新路径并存，并复用 catalog 的既有校验。

### S7【中】proposal:39-40 的结果没有可见性过滤
proposal:39 只 SELECT plugin_id FROM plugin_search。
**代码事实**：search/index.ts:46-47 注释写明「检索先取候选再逐行判可见性」，
review_search 的做法是撤回/隐藏不改索引、**查询时统一解析可见性**（0014:15-16）。
② 后果：若将来 plugin_search 收录了任何非公开来源（或旧值残留，见 S3），
会直接把不该出现的 plugin_id 返回给调用方。
③ 建议：查询仍只当“候选”，结果必须 JOIN 业务表按当前可见性过滤后再返回。

### S8【中】bm25 排序写法未给权重，且 UNINDEXED 列参与权重计算易错
proposal:44 只说按 bm25 排序，未给任何列权重。
**代码事实**：search/index.ts:49-53 的做法是显式权重 `bm25(review_search, 0.0, 8.0, 1.0, 4.0)`，
第 1 个 0.0 正对应 UNINDEXED 列，并注释说明“标题 8 / 正文 1 / 插件名 4”的取舍。
② 后果：不显式给权重时 bm25 对 UNINDEXED 列取值依赖实现细节，排序结果不可预期。
③ 建议：照 0014 的写法显式给 `bm25(plugin_search, 0.0, 8.0, 1.0)`。

### S9【低】方案未提 q 的长度上限，存在无界输入/慢查询面
proposal:35 无任何 q 校验。对照：search/index.ts:137 `.slice(0,100)`、
plugins/search/route.ts:20 `.slice(0, 100)`、catalog maxLength 100。
② 后果：超长 q 会造成不必要的索引扫描（proposal:48 期望 <50ms 会被打破），也是注入放大面。
③ 建议：统一 slice(0,100)，与全站一致。

### S10【低】“不新建表”的取舍未说明，与 0014 的独立 FTS 表理由不冲突但需写明
proposal:12 用独立 FTS 表（非 external content）——**这个选择与 0014:11-14 的理由一致**，是对的：
plugin 的展示名/简介跨 plugins + plugin_field_overrides 两张表，external content 不适用。
建议：把这条理由写进方案，避免后续维护者误改成 external content。

## 一句结论
方案方向（FTS5 + bm25）没问题，但**四处硬伤**：SQL 拼接注入（S1，实测可逃逸）、
中文搜索实际不可用（S2，实测 0 命中）、索引值与展示值不一致（S3）、
索引写入非事务（S4）。另外它**覆盖了一个已存在且更安全的同名接口**（S6）。
本仓 migrations/0014 + modules/search 已经给出正确样板，建议直接照抄而不是另起一套。