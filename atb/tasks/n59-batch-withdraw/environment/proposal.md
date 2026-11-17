# 方案：批量撤回内容

## 背景
作者想一次撤回多篇评测。需要批量接口。

## 技术方案

### 1. 接口
```ts
POST /api/me/reviews/batch-withdraw
body: { reviewIds: string[] }

for (const id of reviewIds) {
  db.prepare(
    "UPDATE content_entities SET withdrawn_at = ? WHERE id = ?"
  ).run(Date.now(), id);
}
```

### 2. 广播
撤回后通知相关订阅者。

## 预期
- 一次撤多篇
- 减少操作次数
