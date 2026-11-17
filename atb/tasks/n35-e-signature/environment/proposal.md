# 方案：用户协议签署与留证

## 背景
需要用户同意服务条款，且要能证明「他签过」。

## 技术方案

### 1. 协议表
```sql
CREATE TABLE agreement_signatures (
  id          TEXT PRIMARY KEY,
  user_id     TEXT NOT NULL REFERENCES users(id),
  agreement   TEXT NOT NULL,
  version     TEXT NOT NULL,
  signed_at   INTEGER NOT NULL
);
```

### 2. 签署流程
用户首次登录后展示协议，点击「同意」即写入：

```ts
await db.prepare(
  "INSERT INTO agreement_signatures (id, user_id, agreement, version, signed_at) VALUES (?, ?, ?, ?, ?)"
).run(newId(), ctx.userId, 'terms', '1.0', Date.now());
```

### 3. 留证
记录用户 ID、协议版本、签署时间。

## 预期
- 能证明用户同意过
- 协议更新后可重新签署
