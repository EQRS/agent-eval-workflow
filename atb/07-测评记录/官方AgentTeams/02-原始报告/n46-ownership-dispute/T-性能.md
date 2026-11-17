# N-46 评审报告 · T-性能工程师（Agent Teams 复用）

> 测评：官方 Agent Teams · 成员 b-e1-b（复用）
> 日期：2026-11-17

## 摘要（前 5 条）

1. 【严重】第 26 行「直接改 plugin_ownerships 的归属人」会撞上部分唯一索引 uq_plugin_ownerships_verified，写入必然失败重试
   - 依据：migrations/0040_user_roles.sql:119-120
   - 后果：不是「慢」，而是任何未先释放旧 verified 的更新都会抛约束错误、事务回滚、上层重试
   - 建议：三步原子写（旧行置 rejected → 新行置 verified → 写审计）

2. 【严重】第 16 行 status 无 CHECK、也无索引，管理员队列查询会全表扫描
   - 依据：proposal.md:12-19 无 CREATE INDEX；对照组 plugin_ownerships 有 3 条索引（0040:113-115）
   - 建议：补 idx_ownership_disputes_queue(status, created_at, id)

3. 【严重】第 23 行「任何用户可对任意插件提争议」缺唯一约束，单用户即可刷爆
   - 依据：现有表有 UNIQUE(plugin_id, user_id)（0040:111），方案没有
   - 建议：加复合唯一 + 部分唯一索引限制每插件最多一个 pending

4. 【中】evidence TEXT 无长度上限
   - 依据：对照 claimed_note 有 CHECK(length <= 500)（0040:89）
   - 后果：SQLite 无部分读取，1MB 文本整行读进内存

5. 【中】created_at 缺非负 CHECK，且无归档策略
   - 依据：对照组所有时间列带 CHECK(created_at >= 0)

## 其余

6. 缺 reviewer_id / reviewed_at
7. 未说明改归属是否在事务内 → 会留下「插件无归属」中间态
8. 外键未声明 ON DELETE 行为