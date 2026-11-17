# 方案：收藏夹条目排序并发更新

## 背景
用户拖拽调整收藏夹顺序时，需要批量更新 `plugin_favorites` 的 `position`。
现在前端每次拖拽都发一个请求，并发时会出现顺序错乱。

## 技术方案

### 1. 表结构
```sql
ALTER TABLE plugin_favorites ADD COLUMN position INTEGER NOT NULL DEFAULT 0;
ALTER TABLE plugin_favorites ADD COLUMN version INTEGER NOT NULL DEFAULT 0;
```

### 2. 更新逻辑
```ts
while (true) {
  const row = db.prepare(
    'SELECT position, version FROM plugin_favorites WHERE user_id = ? AND plugin_id = ?'
  ).get(userId, pluginId);
  const result = db.prepare(
    'UPDATE plugin_favorites SET position = ?, version = version + 1 WHERE user_id = ? AND plugin_id = ? AND version = ?'
  ).run(newPosition, userId, pluginId, row.version);
  if (result.changes === 1) break;
}
```

### 3. 重试
并发冲突时循环重试，直到成功。

## 预期
- 顺序不会错乱
- 并发高时自动重试
- 用户感知不到冲突
