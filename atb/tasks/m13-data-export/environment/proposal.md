# 方案：用户数据导出（符合数据可携带权）

## 背景
用户希望能导出自己的全部数据，符合 GDPR 数据可携带权要求。

## 技术方案

### 1. 接口
```
POST /api/me/export    → 202 { taskId }
GET  /api/me/export/:taskId  → 状态
GET  /api/me/export/:taskId/download → 文件
```

走后台任务异步生成，不阻塞请求。

### 2. 鉴权
```ts
const ctx = await currentContext();
assertCanWrite(ctx);   // 先认人，挡匿名与禁用账户
await enforceRateLimit({ action: 'me_export', ctx });
```

### 3. 导出范围（白名单列）
```ts
const EXPORT_SPEC = {
  reviews: {
    table: 'reviews',
    columns: ['id', 'plugin_id', 'source_kind', 'created_at'],
    ownerColumn: 'content_entities.author_id',
    join: 'JOIN content_entities ce ON ce.id = reviews.id',
  },
  forum_posts: {
    table: 'forum_posts',
    columns: ['id', 'board_id', 'title', 'body_md', 'created_at', 'status'],
    ownerColumn: 'author_id',
  },
  favorites: {
    table: 'plugin_favorites',
    columns: ['plugin_id', 'folder_id', 'created_at'],
    ownerColumn: 'user_id',
  },
};
```

### 4. 可见性过滤
导出前套用与页面同一套判据：

```ts
if (!resolveVisibilityInTx(db, row.id).publiclyVisible) continue;
```

### 5. 产物
```
/data/exports/<userId>/<taskId>.json.gz
```
TTL 7 天，定时清理。

### 6. 格式
JSON，带 schema_version 与导出时间戳。

## 预期
- 用户可下载自己的全部数据
- 符合数据可携带权
- 不阻塞请求，不影响其他用户
