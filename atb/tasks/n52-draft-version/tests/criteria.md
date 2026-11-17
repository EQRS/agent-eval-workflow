# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 基础分 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **save_version 已有，但方案没提 payload_schema_version**
  `content_drafts` 有两个版本列：
  `save_version INTEGER DEFAULT 0`（保存次数）和
  `payload_schema_version INTEGER DEFAULT 1`（数据结构版本）。
  方案只用了 save_version。若 payload 结构升级了，旧客户端的 payload 会覆盖新的。

- [ ] **检查与更新不是原子的**
  §1 的 `if (expectedVersion !== current.save_version)` 是一次 SELECT，
  后面的 UPDATE 是另一次语句。**中间可能被并发插入**——
  两个客户端同时读到 version=5，都通过检查，都 UPDATE → version 变成 7，
  但**其中一方的修改被覆盖**。

### L4 系统性
- [ ] **草稿是软删除，冲突检查没考虑 deleted_at**
  项目 `0018_draft_soft_delete.sql` 给草稿加了 `deleted_at`。
  用户删除草稿后，另一个设备的自动保存会**把已删草稿复活**吗？
  取决于 UPDATE 是否带 `deleted_at IS NULL` 条件——方案没有。

- [ ] **30 秒自动保存会在后台开大量写事务**
  每个编辑中的用户每 30 秒一次 UPDATE。项目 `synchronous=FULL`，
  **每次都是一个 fsync**。100 个用户 = 每秒 3.3 次 fsync。

- [ ] **冲突提示"丢失本地修改"是产品决策，不是技术结论**
  §3 说冲突时「提示用户刷新，丢失本地修改」。
  但项目有 `payload` 快照——本可以**保留两版让用户选**。
  方案直接选择丢弃，是**没有实现的偷懒**。

- [ ] **草稿的 owner_id 是唯一权限边界，方案没校验**
  保存时若只按 `draftId` 更新、不校验 `owner_id`，
  任何登录用户都能覆盖他人草稿。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
