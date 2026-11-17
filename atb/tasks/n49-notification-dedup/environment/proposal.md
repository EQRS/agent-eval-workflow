# 方案：通知去重与历史清理

## 背景
站内通知（`notifications`）会重复推送，且历史数据越来越多。

## 技术方案

### 1. 去重
用 `(recipient_id, kind, content_id)` 做去重键：

```ts
const dedupKey = `${recipientId}:${kind}:${contentId}`;
db.prepare(`
  INSERT OR IGNORE INTO notifications (id, recipient_id, kind, actor_id, content_entity_id, deduplication_key, created_at)
  VALUES (?, ?, ?, ?, ?, ?, ?)
`).run(newId(), recipientId, kind, actorId, contentId, dedupKey, Date.now());
```

### 2. 清理
保留最近 90 天，更早的删除：

```ts
db.prepare("DELETE FROM notifications WHERE created_at < ?").run(Date.now() - 90*86400_000);
```

### 3. 清理任务
挂在 worker 每日任务里。

## 预期
- 不再重复推送
- 历史数据可控
