# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **撤回必须同时递增 visibility_control_version**
  `content_entities` 的撤回不是简单 UPDATE：
  `migrations/0002_triggers.sql` 有触发器要求
  `UPDATE withdrawn_at` 时**必须同时更新 `visibility_control_version`**。
  只写 `withdrawn_at` 会被 ABORT。

- [ ] **withdrawn_at 有配套的 hidden_at / hidden_by / hidden_reason 三列约束**
  `migrations/0001_baseline.sql` 的 CHECK 要求三列同空或同填。
  方案只写 withdrawn_at，需要确认 withdrawn_at 是否有类似的成对约束。

- [ ] **content_entities.id 就是 reviews.id**
  `migrations/0001_baseline.sql`：`reviews.id REFERENCES content_entities(id)`，
  `content_entities` 有 `CHECK (review_id = id)`。
  方案用 `content_entities.id` 是对的，但**没有校验 owner**——
  任何人传别人的 reviewId 就能撤回。

### L4 系统性
- [ ] **撤回后搜索索引与列表缓存不会自动更新**
  项目 `0014_review_search.sql` 明写「撤回/隐藏不改索引，可见性在查询时解析」。
  方案撤回后**读路径会自动过滤**（这是对的），
  但若方案同时「通知订阅者」，订阅者收到的通知会**指向已撤回内容**。

- [ ] **没有校验当前状态，重复撤回会覆盖 withdrawn_at**
  §1 的 UPDATE 无 `WHERE withdrawn_at IS NULL` 条件。
  第二次撤回会把时间戳改成新值，**丢失原始撤回时间**。

- [ ] **批量操作没有事务，部分失败会留下半状态**
  §1 的 for 循环**没有事务**。若第 5 个失败，前 4 个已撤回。
  在 `synchronous=FULL` 下，每次 UPDATE 都是一次 fsync——
  **100 篇 = 100 次 fsync**。

- [ ] **没有上限，可被用来做拒绝服务**
  `reviewIds` 无长度限制。传 10 万个 id 会长时间持锁。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
