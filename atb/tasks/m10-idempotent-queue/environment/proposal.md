# 方案：插件发现任务的幂等去重改造

## 背景
插件发现任务（plugin_discovery）目前按天幂等入队：dedup key = `daily:${date}`。
但实际运行中发现，同一天被多次触发的发现任务会导致重复调用 GitHub API。

## 技术方案

### 1. 复用现有队列
不新建表，直接在 background_tasks 上做。任务类型已是 `plugin_discovery`（0007 迁移已放开）。

### 2. 新的 dedup key 设计
把去重粒度从「天」细化到「数据源」：

```ts
export function discoveryDedupKey(source: string, date: string): string {
  return `discovery:${source}:${date}`;
}

await tasks.enqueue({
  taskType: 'plugin_discovery',
  deduplicationKey: discoveryDedupKey(source, ymd(now)),
  requestedBy: null,
  payload: { source, date: ymd(now) },
});
```

这样每个数据源每天一个任务，互不干扰。

### 3. 手动触发
管理员可以手动触发某个数据源的发现：

```
POST /api/admin/discovery/trigger
body: { source: "github" }
```

实现：用 `discoveryDedupKey(source, ymd(now))` — 若当天已有任务，enqueue 会因 UNIQUE 冲突返回已存在的任务（幂等）。

### 4. 重试
沿用队列的 RETRY_BACKOFF_MS = [30s, 120s]、MAX_RECOVERABLE_ATTEMPTS = 3。

## 预期
- 每个数据源每天只跑一次
- 管理员手动触发不会产生重复任务
- 复用现有队列，无新表
