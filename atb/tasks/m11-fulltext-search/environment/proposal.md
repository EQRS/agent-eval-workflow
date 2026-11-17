# 方案：插件全文搜索功能

## 背景
插件数量增加，用户需要按关键词搜索插件名称和描述。

## 技术方案

### 1. 搜索实现
使用 SQLite FTS5 虚拟表：

```sql
CREATE VIRTUAL TABLE plugin_search USING fts5(
  plugin_id UNINDEXED,
  name,
  description
);
```

### 2. 索引维护
插件新增或更新时同步写入：

```ts
export async function upsertSearchIndex(plugin) {
  await db.prepare(
    "DELETE FROM plugin_search WHERE plugin_id = ?"
  ).run(plugin.id);
  await db.prepare(
    "INSERT INTO plugin_search (plugin_id, name, description) VALUES (?, ?, ?)"
  ).run(plugin.id, plugin.name, plugin.description);
}
```

### 3. 查询接口
```
GET /api/plugins/search?q=<关键词>
```

```ts
const sql = `SELECT plugin_id FROM plugin_search WHERE plugin_search MATCH '${q}'`;
const rows = db.prepare(sql).all();
```

### 4. 排序
按 FTS5 的 bm25 分数排序，相关度高的在前。

## 预期
- 支持中文与英文搜索
- 响应时间 < 50ms
