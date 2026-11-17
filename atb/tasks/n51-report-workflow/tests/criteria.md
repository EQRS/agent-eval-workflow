# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 基础分 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **reports 的 target_kind 是 CHECK 闭集，且它决定 content_entity_id 的语义**
  `migrations/0012_interactions.sql`：
  `target_kind IN ('review','comment','pack_release')`，而 `content_entity_id` 是 NOT NULL。
  **举报"评论"时 content_entity_id 指向的是评论所在的评测，不是评论本身**——
  方案的 SQL 只按 `content_entity_id` 计数，会把「评论被举报 3 次」算成
  「这条评测被举报 3 次」，**误隐藏整篇评测**。

- [ ] **reports 有 comment_id 列，但方案没用**
  举报评论时，真正被举报对象是 `comment_id`。

- [ ] **state 是 TEXT 闭集（open/resolved/dismissed）**
  方案把批量改成 `resolved`，但**被自动隐藏的内容应该是 dismissed 还是 resolved**？
  语义上「举报成立」才是 resolved。

### L4 系统性
- [ ] **通知作者 = 反向暴露举报人**
  方案 §2 通知作者「你的内容因举报被隐藏」。
  若作者是恶意内容发布者，他会据此推断**是谁举报的**（尤其举报量小时）。
  应向作者隐藏「因举报」这个信息，只说「内容违规」。

- [ ] **自动隐藏没有申诉入口**
  项目 `content_entities` 的 `hidden_at` 由管理员操作（有审计）。
  自动隐藏**没有 actor**——事后无法区分"管理员隐藏"和"系统自动隐藏"。

- [ ] **阈值可被刷**
  任何用户可以对同一内容举报 1 次。若攻击者用 3 个账号举报正常内容，
  **就能自动隐藏它**。方案没有任何「同一人不能重复举报」的约束
  （`reports` 表没有 `UNIQUE(reporter_id, content_entity_id)`）。

- [ ] **同步执行会让举报接口变慢**
  §3 说「在举报接口处理完后同步执行」。
  项目所有写接口都在同步驱动下（`db.ts`），多一次 COUNT + UPDATE 会拉长响应。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
