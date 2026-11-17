# 方案：插件更新时推送 Webhook

## 背景
想让第三方服务订阅插件更新事件，需要 Webhook 推送。

## 技术方案

### 1. 订阅配置
用户可配置自己的回调 URL：

```sql
CREATE TABLE webhook_subscriptions (
  id         TEXT PRIMARY KEY,
  user_id    TEXT NOT NULL REFERENCES users(id),
  url        TEXT NOT NULL,
  secret     TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
```

### 2. 推送
插件更新时，异步 POST 到订阅的 URL：

```ts
await fetch(sub.url, {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify(event),
});
```

### 3. 重试
失败重试 3 次。

## 预期
- 第三方能及时收到事件
- 失败可重试
