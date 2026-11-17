# 方案：把事务封装成 async 函数

## 背景
现在写事务要传回调，不够直观。想改成 await 风格：

```ts
// 现在
withTransaction((tx) => {
  tx.db.prepare("...").run();
});

// 想要
await db.transaction(async (tx) => {
  await tx.insert(...);
  await tx.update(...);
});
```

## 技术方案

### 1. 封装
```ts
export async function transaction<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
  db.exec("BEGIN");
  try {
    const result = await fn(tx);
    db.exec("COMMIT");
    return result;
  } catch (e) {
    db.exec("ROLLBACK");
    throw e;
  }
}
```

### 2. 改造调用点
把所有 withTransaction 调用改成 await 风格。

## 预期
- 代码更直观
- 支持异步操作
