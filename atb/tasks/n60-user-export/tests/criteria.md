# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **reviews 表没有 author_id 列**
  `migrations/0001_baseline.sql`：
  `reviews` 只有 `id/plugin_id/source_kind/published_revision_id/latest_submitted_revision_id`。
  **作者在 `content_entities.author_id`**，且 `reviews.id = content_entities.id`。
  §1 的 SQL 会报 "no such column: author_id"。

- [ ] **comments 的字段名需要核实**
  `comments` 表的作者列是 `author_id` 还是别的名字，需确认。

### L4 系统性
- [ ] **导出会带走「已撤回」和「已隐藏」的内容**
  用户的评测被撤回（`withdrawn_at`）或隐藏（`hidden_at`）后，
  作者**仍能看到自己的内容**（这是合理的），
  但导出时若不过滤，会**把管理员隐藏的内容也导出**。
  若隐藏原因是违规，导出等于**帮用户保留了违规证据的副本**。

- [ ] **导出无速率限制，可被用来拖库**
  §1 一次性 SELECT 全部数据。若用户有 10 万条评论，
  会在 `synchronous` 同步驱动下**阻塞事件循环**数秒。
  且项目 `rate-limit.ts` 的动作枚举里**没有 export 动作**。

- [ ] **导出的数据包含他人信息**
  `comments` 可能含回复关系（`reply_to_comment_id`）。
  导出自己的评论时，若带上被回复者的内容片段，就**泄露了他人数据**。

- [ ] **没有审计**
  数据导出是**高价值操作**（等于全量拿走）。
  项目 `operation_records` 是审计载体，方案**没有写审计**。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
