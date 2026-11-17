# 判据清单（agent 不可见）

> **评分结构（v2）**
> - 前提核实：0.5 分
> - **预设考点**：权重 **60%**
> - **超纲发现**：权重 **40%**（域外知识 1.0 / 对抗性 0.8 / 跨系统 0.6 / 代码内 0.3）

## 前提核实（基础项，0.5 分）

- [ ] **方案假设的东西在本仓库是否真实存在？**
  核对表/列/接口/依赖是否在 `migrations/`、`src/`、`package.json` 中真实存在。
  **注意区分**：方案声明新建的（有 CREATE TABLE / ALTER / 明确说「引入」）属合理；
  方案在「**## 现状**」小节声称的，必须是仓库真实情况。

## 必须发现（每项 1 分）

### 跨系统（真实约束）
- [ ] **C-1 领取逻辑不是按 SQL 排序那么简单**
  `tasks/service.ts` 的 claim 逻辑有**租约 + CAS 写回**：
  领取时要 `UPDATE ... WHERE processing_state = 'pending' AND id = ?` 做乐观锁。
  只改 `ORDER BY` 不会改变「谁能领到」，还要看 CAS 是否失败重试。

- [ ] **C-2 现有排序依赖索引，加 priority 会让索引失效**
  项目有 `idx_tasks_due` 之类的索引（按 `available_at, created_at`）。
  改成 `ORDER BY priority DESC, available_at, created_at` 后，
  **原索引的第一列不匹配**，会退化为全表扫描 + 临时 B-Tree 排序。

- [ ] **C-3 优先级会导致「饥饿」**
  §3 说发布 = 100、发现 = 0。若发布任务持续不断，
  **发现任务永远排不上**——这正是「优先级」的经典陷阱。
  项目 `scheduled.ts` 的每日发现如果饿死，目录会停止更新。

### 对抗性
- [ ] **S-1 priority 可被设为任意值**
  §1 只有 `DEFAULT 0`，没有 `CHECK` 也没有枚举。
  任何代码路径都能写 `priority = 999999`，**优先级机制形同虚设**。

### 性能
- [ ] **P-1 加列会让 background_tasks 的所有行变宽**
  `background_tasks` 是高频写入表（心跳、续期、扫描）。
  `ALTER TABLE ADD COLUMN` 在 SQLite 里会让每条记录增加字节，
  且项目 `synchronous=FULL` 下写放大明显。

- [ ] **P-2 priority DESC 排序无法用「前缀索引」**
  若要让 `ORDER BY priority DESC, available_at` 走索引，
  需要建 `INDEX(priority DESC, available_at, created_at)`。
  但 priority 只有 0 和 100 两个值——**索引选择性极差**，几乎无用。


## 超纲发现（权重 40%）

- [ ] **是否发现了预设之外、但经核实成立的问题？**
- [ ] **是否指出「方案要解决的问题已经解决了」？**（若适用，0.5 分）
