# 方案：外部 API 调用的统一重试

## 背景
调用 GitHub API 时会失败，需要统一重试。

## 技术方案

### 1. 重试封装
```ts
export async function withRetry<T>(fn: () => Promise<T>, times = 3): Promise<T> {
  for (let i = 0; i < times; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === times - 1) throw e;
      await sleep(1000 * Math.pow(2, i));  // 1s, 2s, 4s
    }
  }
}
```

### 2. 使用
```ts
const data = await withRetry(() => fetchGitHub(url));
await db.prepare("UPDATE plugins SET description = ? WHERE id = ?").run(data.desc, id);
```

## 预期
- 失败自动重试
- 减少人工干预
