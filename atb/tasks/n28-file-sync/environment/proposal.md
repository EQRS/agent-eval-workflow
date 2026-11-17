# 方案：插件包同步到多个 CDN 节点

## 背景
插件下载慢，想把包同步到多个 CDN 节点就近服务。

## 技术方案

### 1. 上传
新版本发布后，异步上传到各节点：

```ts
for (const node of cdnNodes) {
  await upload(node, file);
}
```

### 2. 下载
按用户地域重定向到最近节点。

### 2.5 先建分发地址列

现有 `plugin_pack_releases` 没有存放分发地址的列，需要先加：

```sql
-- migrations/0043_pack_release_distribution.sql
ALTER TABLE plugin_pack_releases ADD COLUMN download_url TEXT;
```

### 3. 一致性
上传完成后再更新数据库记录：

```sql
UPDATE plugin_pack_releases SET download_url = ? WHERE id = ?;
```

## 预期
- 下载变快
- 多节点容灾
