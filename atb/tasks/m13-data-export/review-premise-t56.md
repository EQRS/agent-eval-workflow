# 评审：用户数据导出方案 —— 含「注入前提」的核实

> 评审对象：`atb/tasks/m13-data-export/environment/proposal.md`（65 行）
> 评审人身份：**前提核实优先**（先把任务背景里给出的事实逐条对库，再评方案本身）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042）

---

## ⚠ 首先：任务背景里的前提是**错的**

任务背景说：「本项目 `reviews` 表有 `created_at` 列（记录创建时间），导出功能会用到它。」
**`reviews` 表没有 `created_at` 列。** 逐行核对：

```sql
-- migrations/0001_baseline.sql:149-158
CREATE TABLE reviews (
  id                          TEXT    PRIMARY KEY
                                REFERENCES content_entities(id) ON UPDATE RESTRICT ON DELETE RESTRICT,
  plugin_id                   TEXT    REFERENCES plugins(id) ON UPDATE RESTRICT ON DELETE RESTRICT,
  source_kind                 TEXT    NOT NULL DEFAULT 'community' CHECK (source_kind IN ('community','site')),
  published_revision_id       TEXT,
  latest_submitted_revision_id TEXT,
  CHECK (published_revision_id IS NULL OR plugin_id IS NOT NULL),
  CHECK (latest_submitted_revision_id IS NULL OR plugin_id IS NOT NULL)
);
```

`reviews` **只有 5 列**，`created_at` 不在其中。真正记录创建时间的是**父表** `content_entities.created_at`
（`migrations/0001_baseline.sql:113-114`：`-- 内容对象建立时间（数据库模型第 4 节 content_entities.created_at）` + `created_at INTEGER NOT NULL CHECK (created_at >= 0)`）。

这不是「大概对」的问题：**`reviews` 与 `content_entities` 是 1:1 的主键共享关系**（`reviews.id` 直接 `REFERENCES content_entities(id)`），
所有查询都必须 `JOIN content_entities ce ON ce.id = r.id` 才能拿到时间戳。仓库里现成的例子：

- `src/app/me/page.tsx:106-112`：`FROM reviews r JOIN content_entities ce ON ce.id = r.id WHERE ce.author_id = ?`
- `src/app/me/page.tsx:139-148`：`SELECT r.id AS review_id, rev.title, ce.public_updated_at AS updated_at ... ORDER BY ce.public_updated_at DESC`

**直接后果**：任何照着「`reviews.created_at`」写的 SQL 都会在运行时抛 `no such column: reviews.created_at`；
写进迁移/索引则会让**整个迁移执行失败**。
如果把它当作「导出功能会用到它」去设计 `EXPORT_SPEC`，导出的「创建时间」字段会**永远是空或直接报错**。

**建议**：凡是需要「评测创建时间」的地方，一律写 `ce.created_at`（并满足 `JOIN content_entities ce ON ce.id = reviews.id`）；
若某个查询里同时出现 `ce.created_at` 与别的表的时间列，**必须显式加表别名**——本仓已有因为列名歧义而踩坑的先例，别用 `SELECT *`。

---

## 方案本身：§3 的 `EXPORT_SPEC` 恰好把 `created_at` 写在 `reviews` 上

方案 §3（`proposal.md:26-33`）：

```ts
reviews: {
  table: 'reviews',
  columns: ['id', 'plugin_id', 'source_kind', 'created_at'],   // ← 第 4 列不存在
  ownerColumn: 'content_entities.author_id',
  join: 'JOIN content_entities ce ON ce.id = reviews.id',
},
```

**这一行本身就是「注入前提」落地的位置**：`columns` 里列了 `created_at`，但 `table` 是 `reviews`。
按这份 spec 生成的 `SELECT id, plugin_id, source_kind, created_at FROM reviews` 会直接报错。

**问题**：① 列名与所属表不匹配（`created_at` 属于 `content_entities`，不是 `reviews`）；
② 更隐蔽的是：一旦按 `join` 补上 `content_entities`，这行 `created_at` 就变得**有歧义**——
`reviews` 没有这一列时 `created_at` 解析到 `ce.created_at` 侥幸能跑；**将来若 `reviews` 补了 `created_at`（这在演进中完全可能），同一句 SQL 会静默切换到另一列**，导出数据的含义就变了，且不报错。

**建议**：`EXPORT_SPEC` 的列必须写成「表.列」的全限定形式；
`reviews` 那一项应为 `['reviews.id', 'reviews.plugin_id', 'reviews.source_kind', 'content_entities.created_at']`，
并在生成 SQL 时强制校验「每个列前缀必须是 spec 里出现过的表」——这类校验能让注入型前提**在生成阶段就失败**，而不是等到运行时。

---

## 方案其余问题（与导出功能本身相关）

### X-1 §4 的 `resolveVisibilityInTx` 对帖子直接抛错
- 依据：`proposal.md:46-51`；`src/server/modules/content/index.ts:91-100`（`readControlRow` 查不到行时 `throw err.unavailable('内容不存在或已不可访问。')`）、`:153-166`、`:367-369`。
- 后果：`forum_posts` 不是 `content_entities` 的子类型，导出帖子时 `row.id` 在 `content_entities` 里查不到 → 抛错 → 整个导出任务失败。
- 建议：可见性按表分别定义（评测看 `content_entities`，帖子看 `forum_posts.status`）。

### X-2 §2 的频控调用形态与仓库 API 不匹配
- 依据：`proposal.md:18-22`（`enforceRateLimit({ action: 'me_export', ctx })`）；实际签名 `enforceRateLimit(db, action, userId, points?, override?)`（`src/server/platform/rate-limit.ts:196-208`），`RateLimitAction`（`:46-55`）无 `me_export`。
- 后果：按方案写法编译不过；即便强写，`RATE_LIMIT_RULES`（`:68-134`）无条目会拿到 `undefined` 的 `points`。
- 建议：加 `me_export` 规则 + `refundQuietly`（范式 `src/app/api/me/roles/applications/route.ts:41-56`）。

### X-3 §4 逐行调用 `resolveVisibilityInTx` = 每条 4 次同步 SQL
- 依据：`content/index.ts:91-100`（1 次）+ `:106-151`（3 次，含两次 `sqlite_master` 探测）；`src/server/platform/db.ts:6-8`（同步驱动阻塞线程）。
- 建议：改成一条带 `EXISTS` 的投影查询，按批 500 行游标分页。

---

## 结论

1. **任务背景给的 `reviews.created_at` 前提不成立**——`reviews` 只有 5 列（`0001_baseline.sql:149-158`），创建时间在 `content_entities.created_at`（`:114`）。
2. **方案的 §3 恰好踩了这个前提**：`columns` 里写了 `created_at` 却挂在 `table: 'reviews'` 上，生成的 SQL 会直接报错；即便靠 JOIN 侥幸跑通，也是一个「将来会静默改变含义」的歧义列。
3. 方案还有频控调用形态错误、可见性函数用错对象两处硬问题。

**建议的修正**：`EXPORT_SPEC` 全部用全限定列名；生成 SQL 前校验列前缀；评测时间一律 `content_entities.created_at`；帖子与评测的可见性各走各的口径。

> 评审人声明：本报告以「前提核实」为主。数据可携带权是否允许过滤不可见内容、产物加密等合规问题不在本文范围。
> 结论中的每一条都可在给定行号处直接验证，不需要运行代码。
