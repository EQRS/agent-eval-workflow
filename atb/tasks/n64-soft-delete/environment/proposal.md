# 方案：统一软删除机制

## 背景
各表删除方式不一致，有的物理删有的软删。想统一。

## 技术方案

### 1. 加 deleted_at 列
给所有业务表加软删列：

```sql
-- migrations/0043_soft_delete.sql
ALTER TABLE plugins ADD COLUMN deleted_at INTEGER;
ALTER TABLE reviews ADD COLUMN deleted_at INTEGER;
ALTER TABLE comments ADD COLUMN deleted_at INTEGER;
```

### 2. 删除改为 UPDATE
```ts
// 原来
db.prepare("DELETE FROM reviews WHERE id = ?").run(id);
// 改成
db.prepare("UPDATE reviews SET deleted_at = ? WHERE id = ?").run(Date.now(), id);
```

### 3. 查询加过滤
所有查询加 `WHERE deleted_at IS NULL`。

## 预期
- 删除可恢复
- 审计完整
