# 方案：用户偏好读取加缓存

## 背景
用户偏好（`user_preferences`）每次请求都读库，想加缓存。

## 技术方案

### 1. 进程内缓存
```ts
const cache = new Map<string, Preferences>();

export function getPreferences(userId: string) {
  if (cache.has(userId)) return cache.get(userId)!;
  const row = db.prepare(
    "SELECT * FROM user_preferences WHERE user_id = ?"
  ).get(userId);
  cache.set(userId, row);
  return row;
}
```

### 2. 写入时清缓存
```ts
export function setPreferences(userId: string, prefs: Preferences) {
  db.prepare("UPDATE user_preferences SET ... WHERE user_id = ?").run(userId);
  cache.delete(userId);
}
```

### 3. 定期全清
每小时清空一次缓存。

## 预期
- 偏好读取从查询变成内存
- 首屏更快
