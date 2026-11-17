# 方案：给后台任务加优先级

## 现状
后台任务（`background_tasks`）现在按 `available_at, created_at` 先进先出。
发布类任务和发现类任务混在一起，希望让发布优先。

## 技术方案

### 1. 加优先级列
```sql
-- migrations/0043_task_priority.sql
ALTER TABLE background_tasks ADD COLUMN priority INTEGER NOT NULL DEFAULT 0;
```

### 2. 领取时按优先级排序
```ts
SELECT * FROM background_tasks
WHERE processing_state = 'pending' AND available_at <= ?
ORDER BY priority DESC, available_at ASC, created_at ASC
LIMIT ?
```

### 3. 设置优先级
发布类任务 priority = 100，发现类 = 0。

## 预期
- 发布任务优先执行
- 用户提交后更快公开
