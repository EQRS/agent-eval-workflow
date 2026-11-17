# 方案：给所有写接口加幂等

## 背景
用户重复点击会重复提交，需要幂等。

## 技术方案

### 1. 幂等表
```sql
CREATE TABLE idempotency_keys (
  key         TEXT PRIMARY KEY,
  response    TEXT NOT NULL,
  created_at  INTEGER NOT NULL
);
```

### 2. 中间件
在写接口前统一检查：

```ts
const key = req.headers['idempotency-key'];
if (key) {
  const cached = db.prepare(
    'SELECT response FROM idempotency_keys WHERE key = ?'
  ).get(key);
  if (cached) return jsonOk(JSON.parse(cached.response));
}
// 执行原逻辑
// 成功后写表
```

### 3. 过期
24 小时后清理。

## 预期
- 重复请求返回相同结果
- 用户不会重复提交
