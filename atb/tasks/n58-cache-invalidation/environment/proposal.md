# 方案：插件列表缓存加失效通知

## 背景
插件列表每次请求都查库，想加缓存。数据变更时要失效。

## 技术方案

### 1. 缓存
```ts
let cache: Plugin[] | null = null;
let cachedAt = 0;

export function listPlugins() {
  if (cache && Date.now() - cachedAt < 60_000) return cache;
  cache = db.prepare("SELECT * FROM plugins").all();
  cachedAt = Date.now();
  return cache;
}
```

### 2. 失效
数据变更时主动清缓存：

```ts
export function invalidate() {
  cache = null;
}
```

### 3. 跨进程通知
用 SQLite 的 user_version 做跨进程信号：

```ts
export function invalidate() {
  cache = null;
  db.pragma(`user_version = ${Date.now()}`);
}

export function listPlugins() {
  const v = db.pragma("user_version", { simple: true });
  if (cache && v === cachedVersion) return cache;
  // ...
}
```

## 预期
- 列表查询从 ms 降到 μs
- 跨进程一致
