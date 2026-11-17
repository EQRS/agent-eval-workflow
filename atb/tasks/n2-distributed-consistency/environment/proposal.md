# 方案：settings 读取加进程内缓存

## 背景
`currentContext` 每次请求都会调 `effectiveSettings()`，
而它每次都执行 `SELECT key, value FROM site_settings` 全表扫。
页面渲染一次可能调用多次。

## 技术方案

### 1. 进程内缓存
```ts
let cached: Map<string, string> | null = null;
let cachedAt = 0;

export function getSettings(db) {
  if (cached && Date.now() - cachedAt < 60_000) return cached;
  const rows = db.prepare('SELECT key, value FROM site_settings').all();
  cached = new Map(rows.map(r => [r.key, r.value]));
  cachedAt = Date.now();
  return cached;
}
```

### 2. 失效
管理员改设置后，清掉缓存：

```ts
export async function updateSetting(db, key, value) {
  await db.prepare('INSERT OR REPLACE INTO site_settings (key, value) VALUES (?, ?)').run(key, value);
  cached = null;   // 清缓存
}
```

### 3. 部署
站点当前是单进程部署（`server.mjs` 单 Node 进程），
所以进程内缓存不会有一致性问题。

## 预期
- settings 读取从全表扫变成内存读
- 每次页面渲染省 3-5 次查询
