# 方案：用户数据自助导出

## 背景
用户要求导出自己的数据。需要提供导出功能。

## 技术方案

### 1. 导出接口
```ts
GET /api/me/export

const user = getUser(ctx.userId);
const reviews = db.prepare("SELECT * FROM reviews WHERE author_id = ?").all(ctx.userId);
const comments = db.prepare("SELECT * FROM comments WHERE author_id = ?").all(ctx.userId);

return json({ user, reviews, comments });
```

### 2. 格式
JSON 格式，包含所有数据。

## 预期
- 用户能拿走自己的数据
- 满足数据可携带权
