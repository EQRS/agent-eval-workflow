# 方案：举报的自动处置

## 背景
举报（`reports`）现在全靠人工看。想加自动化：达到阈值的自动隐藏。

## 技术方案

### 1. 阈值规则
同一内容被举报 `>= 3` 次，自动隐藏：

```ts
const count = db.prepare(
  "SELECT COUNT(*) AS c FROM reports WHERE content_entity_id = ? AND state = 'open'"
).get(entityId).c;

if (count >= 3) {
  db.prepare(
    "UPDATE content_entities SET hidden_at = ?, hidden_reason = 'auto_report' WHERE id = ?"
  ).run(Date.now(), entityId);
  db.prepare("UPDATE reports SET state = 'resolved' WHERE content_entity_id = ?").run(entityId);
}
```

### 2. 通知
自动隐藏后通知作者：

```ts
await deliverModerationResultNotification(...);
```

### 3. 挂载
在举报接口处理完后同步执行。

## 预期
- 违规内容快速下架
- 减少人工负担
