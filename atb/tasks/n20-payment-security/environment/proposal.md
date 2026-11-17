# 方案：插件付费下载的订单系统

## 背景
部分作者希望插件能付费下载。需要一套订单系统。

## 技术方案

### 1. 订单表
```sql
CREATE TABLE orders (
  id         TEXT PRIMARY KEY,
  plugin_id  TEXT NOT NULL REFERENCES plugins(id),
  user_id    TEXT NOT NULL REFERENCES users(id),
  amount     INTEGER NOT NULL,
  status     TEXT NOT NULL DEFAULT 'pending',
  created_at INTEGER NOT NULL
);
```

### 2. 支付回调
支付平台回调时更新状态：

```ts
export async function onPaymentCallback(req) {
  const { orderId, tradeNo } = await req.json();
  await db.prepare(
    "UPDATE orders SET status = 'paid' WHERE id = ?"
  ).run(orderId);
  return ok();
}
```

### 3. 下载鉴权
已支付的用户可下载插件包。

## 预期
- 支持付费下载
- 流程简单
