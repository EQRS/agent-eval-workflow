# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **未读数在导航栏，等于每页都查一次**
  项目 `session-nav.tsx` 在**每个页面**渲染。
  §3 的 COUNT 查询会在每次页面渲染时执行——
  全站页面数 × 用户数 = 大量重复查询。

- [ ] **notifications 的索引是 (recipient_id, read_at, created_at DESC, id)**
  这个复合索引**只对 recipient_id 前导有效**。
  §3 的 `WHERE recipient_id = ? AND read_at IS NULL` 能用上它，
  但要确认 `read_at IS NULL` 在索引第二列能有效利用。

### L4 系统性
- [ ] **批量已读没有上限，会一次锁表**
  §2 的 UPDATE 无 `LIMIT`。若有 10 万条未读，
  一次 UPDATE 在 `synchronous=FULL` 下是**长时间持写锁**。
  项目 `rate-limit.ts` 的动作枚举里**没有 notification 动作**。

- [ ] **批量已读没有事务边界与审计**
  §2 是单条 UPDATE，本身是原子的。
  但**没有写 operation_records**——
  用户说"我没点过已读"时无法举证。

- [ ] **read_at 是可改字段，与不可变审计冲突**
  项目的 `operation_records` 有 `BEFORE UPDATE` 触发器。
  `notifications` 没有。若将来要"通知不能改"，
  已读本身就是一个修改——这个矛盾方案没有面对。

- [ ] **分类展示会暴露 kind 的实现细节**
  `kind` 是 CHECK 闭集（如 `comment_on_content`、`reply_to_comment`）。
  直接按 kind 展示，用户看到的是**内部枚举名**。
  项目有 `docs/用词规范.md` 约束对外文案。

- [ ] **未读数与列表可能有竞态**
  §3 的 COUNT 与 §1 的列表查询是**两次独立查询**。
  用户看到"3 条未读"但点进去只有 2 条（第 3 条刚被标记已读）。
  这是可接受的产品行为，但方案没有说明。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
