# 方案：通知中心增加分类与批量操作

## 背景
通知越来越多，用户想按类型筛选与批量已读。

## 技术方案

### 1. 增加分类
按 `kind` 分组展示。

### 2. 批量已读
```ts
POST /api/me/notifications/read-all
// 把当前用户所有未读置为已读
db.prepare(
  "UPDATE notifications SET read_at = ? WHERE recipient_id = ? AND read_at IS NULL"
).run(Date.now(), ctx.userId);
```

### 3. 未读数
导航栏显示未读数：

```ts
const unread = db.prepare(
  "SELECT COUNT(*) AS c FROM notifications WHERE recipient_id = ? AND read_at IS NULL"
).get(userId).c;
```

## 预期
- 用户能管理通知
- 未读数实时
