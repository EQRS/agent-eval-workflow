# 方案：数据保留期与自动过期

## 背景
为了合规，需要给各类数据设置保留期，到期自动删除。

## 技术方案

### 1. 保留期配置
```ts
export const RETENTION = {
  notifications: 90,      // 天
  reports: 365,
  content_drafts: 180,
  operation_records: 365,
};
```

### 2. 过期清理
```ts
for (const [table, days] of Object.entries(RETENTION)) {
  const cutoff = Date.now() - days * 86400_000;
  db.prepare(`DELETE FROM ${table} WHERE created_at < ?`).run(cutoff);
}
```

### 3. 执行
挂 worker 每日任务。

## 预期
- 合规有据可依
- 数据自动清理
