# 方案：把多步写操作合并成事务

## 背景
发布评测时要做多件事：写评测、更新插件统计、发通知。现在分散在多个函数里，担心不一致。

## 技术方案

### 1. 优化前（我们要合并的写法）
三个模块各写各的，不在同一事务：
```ts
export async function publishReview(input) {
  await createReview(input);        // 写 reviews
  await incrementCount(input.pluginId);  // 更新插件统计
  await sendNotification(input.authorId);                 // 发通知
}
```

### 2. 改造
包在一个事务里：

```ts
export async function publishReview(input) {
  return withTransaction(async (tx) => {
    await createReview(input, tx);
    await catalog.incrementCount(input.pluginId, tx);
    await sendNotification(input.authorId, tx);
  });
}
```

## 预期
- 三步要么全成功要么全回滚
- 数据一致性有保障
