# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 基础分 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **真实的去重键已经是 deduplication_key 且有 UNIQUE**
  `migrations/0008_notifications.sql` 已有
  `deduplication_key TEXT NOT NULL` + `UNIQUE (deduplication_key)`。
  方案又造一个 `recipientId:kind:contentId` 的键——**两种键并存**，
  而表上只有原键有唯一约束。

- [ ] **kind 是 CHECK 闭集，方案的新 kind 写不进去**
  `kind TEXT NOT NULL CHECK (kind IN ('comment_on_content','reply_to_comment'))`。
  后续迁移（0041 等）扩过这个闭集。方案若用新 kind，**必须再重建一次表**。

- [ ] **有一列 actor_id 的 CHECK**
  `CHECK (recipient_id <> actor_id)`——"自己不通知自己"。
  方案的 INSERT 若 actorId 等于 recipientId，会**直接违反约束**。

### L4 系统性
- [ ] **清理 90 天会删除「未读」通知**
  项目有 `read_at` 列（`read_at IS NULL` 表示未读）。
  §2 只按 `created_at` 删——**用户 90 天没登录，回来时所有通知都没了**，
  而项目设计上未读是对用户有意义的。应优先保留未读。

- [ ] **清理删通知不删关联数据，但审计会要求可追溯**
  通知是"已发生事件"的记录。删掉后若涉及争议（如"我没收到通知"），
  **无法举证**。项目的 `operation_records` 是不可删的，但通知不是——
  两者对"历史"的态度不一致。

- [ ] **去重键含 recipientId，但表上已有的键不含**
  若同一条内容对同一用户产生两条不同类型通知，
  方案的键会区分，但**若 kind 相同、contentId 相同、不同 actor**，
  方案会认为是同一条——**多个评论者会被合并成一条通知**。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
