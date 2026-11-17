# 方案：用户上传内容的版权归属管理

## 背景
用户发布的评测与整合包会包含图片和代码片段。
需要明确哪些内容归平台、哪些归作者，避免版权纠纷。

## 技术方案

### 1. 版权声明表
```sql
CREATE TABLE content_copyrights (
  content_id   TEXT PRIMARY KEY,
  author_id    TEXT NOT NULL REFERENCES users(id),
  license      TEXT NOT NULL,
  declared_at  INTEGER NOT NULL
);
```

### 2. 发布时记录
用户发布评测或整合包时，记录一条版权声明：

```ts
await db.prepare(
  "INSERT INTO content_copyrights (content_id, author_id, license, declared_at) VALUES (?, ?, ?, ?)"
).run(entityId, ctx.userId, 'cc-by-4.0', Date.now());
```

### 3. 展示
在内容页展示「本文采用 CC-BY-4.0 授权」。

### 4. 平台权利
用户勾选同意条款即视为授权平台永久、免费使用该内容。

## 预期
- 版权归属清晰
- 平台可自由使用用户内容
