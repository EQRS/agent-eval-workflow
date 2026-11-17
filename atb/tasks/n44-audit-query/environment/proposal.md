# 方案：审计记录查询优化

## 现状
后台的审计页面（`/admin/operations`）现在查得很慢，要优化。

## 技术方案

### 1. 加物化统计
```sql
-- migrations/0043_audit_stats.sql
CREATE TABLE audit_stats (
  day        TEXT PRIMARY KEY,
  action     TEXT NOT NULL,
  count      INTEGER NOT NULL DEFAULT 0
);
```

### 2. 写入时累加
每次写审计记录时，同步累加统计：

```ts
recordOperationInTx(db, input);
db.prepare(
  "INSERT INTO audit_stats (day, action, count) VALUES (?, ?, 1) ON CONFLICT(day, action) DO UPDATE SET count = count + 1"
).run(today, input.actionType);
```

### 3. 页面改查统计表
`&{B}
```sql
SELECT * FROM audit_stats WHERE day = ? ORDER BY count DESC LIMIT 10;
```

## 预期
- 审计页面从 5 秒降到毫秒级
