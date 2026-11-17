# 方案：插件推广位投放

## 背景
想给付费作者提供推广位，让他们的插件在列表页靠前展示。

## 技术方案

### 1. 推广位表
```sql
-- migrations/0043_ad_slots.sql
CREATE TABLE ad_slots (
  id          TEXT PRIMARY KEY,
  plugin_id   TEXT NOT NULL REFERENCES plugins(id),
  buyer_id    TEXT NOT NULL REFERENCES users(id),
  starts_at   INTEGER NOT NULL,
  ends_at     INTEGER NOT NULL,
  amount      INTEGER NOT NULL,
  created_at  INTEGER NOT NULL
);
```

### 2. 展示
插件列表页优先展示推广中的插件：

```ts
const promoted = db.prepare(
  "SELECT plugin_id FROM ad_slots WHERE starts_at <= ? AND ends_at > ?"
).all(now, now);
```

### 3. 计费
按展示次数计费，每小时结算一次。

## 预期
- 作者可以购买曝光
- 平台增加收入
