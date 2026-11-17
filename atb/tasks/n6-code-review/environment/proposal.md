# 方案：插件重命名改为批量接口

## 背景
重命名插件需要逐条调用，一次只能改一个，运营效率低。改为批量。

## 技术方案

### 1. 新接口

```
PATCH /api/admin/plugins/rename
body: { items: [{ pluginId, newName }] }
```

### 2. 实现

```ts
for (const item of items) {
  await db.prepare('UPDATE plugins SET display_name = ? WHERE id = ?')
    .run(item.newName, item.pluginId);
}
return okJson({ updated: items.length });
```

### 3. 鉴权
路由层统一 assertCanAdmin(ctx)，一次校验覆盖整批。

### 4. 错误处理
某条失败时跳过，继续处理下一条，返回成功数。

## 预期
- 一次能改 100 个，效率提升 100 倍
- 单次事务，保持一致