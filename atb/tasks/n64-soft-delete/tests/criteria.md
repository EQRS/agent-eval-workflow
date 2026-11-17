# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？（方案显式声明新建的属合理）

## 必须发现（L3 层）

### L3 跨模块
- [ ] **reviews 表没有 deleted_at 需要的 author_id**
  `reviews` 表与 `content_entities` 同主键。
  软删应该在**哪张表**加列？方案的 `reviews` 加列意义不明。

- [ ] **加列后原来的唯一约束会挡住「删除后重建」**
  项目 `content_drafts` 已有软删（`0018_draft_soft_delete.sql`），
  且有 `uq_drafts_review` **部分唯一索引**（`WHERE deleted_at IS NULL`）。
  方案给其他表加 `deleted_at` 时，**没有同步改唯一索引**——
  软删后再创建同名/同键记录会**撞唯一约束**。

### L4 系统性
- [ ] **软删与硬删的语义完全不同，统一会丢失语义**
  项目已有多种"删除"：
  - `content_entities.withdrawn_at`（撤回，可恢复）
  - `content_entities.hidden_at`（隐藏，管理员操作）
  - `content_drafts.deleted_at`（草稿软删）
  方案的 `deleted_at` 是**第四种**。四者混用会让
  `WHERE deleted_at IS NULL` 的查询**无法表达**"已撤回但未删除"。

- [ ] **外键的 ON DELETE 策略全部失效**
  项目大量 `ON DELETE RESTRICT` 依赖**物理删除**来触发约束。
  改成软删后，**约束永远不触发**——
  删父行不会报错，子行会变成**指向已软删父行的孤儿**。

- [ ] **所有查询都要加过滤，漏一处就是数据泄露**
  项目 `src/` 下有数百处 SELECT。
  方案说"所有查询加 `WHERE deleted_at IS NULL`"——
  **没有机制保证不漏**。项目已有 `resolveVisibility` 作为统一可见性判据，
  方案没有复用它。

- [ ] **软删的数据会永久占用唯一索引空间**
  即使加了 `WHERE deleted_at IS NULL` 的部分索引，
  软删的行仍占用**表空间与索引页**。方案的"预期：删除可恢复"
  没有提到**何时真正清理**——这些行会永久累积。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
