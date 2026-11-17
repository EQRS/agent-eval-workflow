# 方案：定时任务的调度时间改造

## 背景
现有 ensureDailyDiscoveryTask 用「每次启动都试着排一次 + 按天 dedup key」，
这保证了幂等，但实际执行时刻不稳定——取决于 worker 什么时候启动。

产品希望「每天固定时间跑」。

## 技术方案

### 1. 引入 scheduled_at
给 background_tasks 加一列（迁移 0042）：

```sql
ALTER TABLE background_tasks ADD COLUMN scheduled_at INTEGER;
CREATE INDEX idx_tasks_scheduled ON background_tasks(processing_state, scheduled_at, id);
```

### 2. 调度器
worker 里跑一个轻量调度循环（复用现有主循环的节拍）：

```ts
const schedules = [
  { name: 'daily_discovery', at: '03:00', dedup: (d) => `discovery:github:${d}` },
  { name: 'cache_warmup', at: '04:00', dedup: (d) => `warmup:${d}` },
];

async function tick(now: number) {
  const today = ymd(now);
  for (const s of schedules) {
    const target = atTime(now, s.at);
    if (now >= target) {
      await tasks.enqueue({
        taskType: s.name,
        deduplicationKey: s.dedup(today),
        requestedBy: null,
        payload: { date: today },
      });
    }
  }
}
```

### 3. 新增任务类型
cache_warmup 需要新的 task_type（迁移里放开 CHECK）。

## 预期
- 任务在每天固定时刻执行
- 复用现有队列的幂等与重试
- 无新表
