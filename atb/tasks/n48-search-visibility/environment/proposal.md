# 方案：搜索结果加可见性过滤

## 背景
全文搜索（`review_search`）会用 FTS5 检索评测正文。
需要确保**已撤回/已隐藏**的评测不出现在搜索结果里。

## 技术方案

### 1. 检索时过滤
```ts
const rows = db.prepare(`
  SELECT review_id FROM review_search
  WHERE review_search MATCH ?
  LIMIT ?
`).all(query, limit);

// 再逐条判断可见性
const visible = rows.filter(r => isVisible(r.review_id));
```

### 2. 索引重建
内容变更时同步更新 FTS：

```ts
db.prepare("DELETE FROM review_search WHERE review_id = ?").run(id);
db.prepare("INSERT INTO review_search (review_id, title, body, plugin_name) VALUES (?,?,?,?)")
  .run(id, title, body, pluginName);
```

### 3. 撤回时删除索引
撤回内容时直接从 FTS 删除：

```ts
db.prepare("DELETE FROM review_search WHERE review_id = ?").run(id);
```

## 预期
- 搜索不再返回已撤回内容
- 索引与内容一致
