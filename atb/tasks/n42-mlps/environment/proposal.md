# 方案：等保二级合规改造

## 背景
平台要满足等级保护二级要求，需要补充安全审计与访问控制。

## 技术方案

### 1. 审计日志
```sql
-- migrations/0043_audit_log.sql
CREATE TABLE audit_log (
  id         TEXT PRIMARY KEY,
  user_id    TEXT,
  action     TEXT NOT NULL,
  ip         TEXT,
  created_at INTEGER NOT NULL
);
```

### 2. 记录点
所有写操作都记一条：

```ts
db.prepare(
  "INSERT INTO audit_log (id, user_id, action, ip, created_at) VALUES (?, ?, ?, ?, ?)"
).run(newId(), ctx.userId, action, clientIp, Date.now());
```

### 3. 访问控制
管理员操作需要二次密码确认。

### 4. 日志保留
保留 6 个月。

## 预期
- 满足等保二级审计要求
- 有据可查
