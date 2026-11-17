# 方案：作者会员自动续费

## 背景
想给作者提供会员服务：按月付费，可享受更高曝光。
默认自动续费，方便用户。

## 技术方案

### 1. 订阅表
```sql
-- migrations/0043_subscriptions.sql
CREATE TABLE subscriptions (
  id           TEXT PRIMARY KEY,
  user_id      TEXT NOT NULL REFERENCES users(id),
  plan         TEXT NOT NULL,
  auto_renew   INTEGER NOT NULL DEFAULT 1,
  next_charge  INTEGER NOT NULL,
  created_at   INTEGER NOT NULL
);
```

### 2. 扣费
每日扫描 `next_charge <= now` 的订阅，自动扣费并顺延一个月。

### 3. 取消
用户可在设置页关闭自动续费。

## 预期
- 续费率提升
- 用户省心
