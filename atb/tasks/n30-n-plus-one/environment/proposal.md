# 方案：插件列表页的分类查询优化

## 背景
插件目录页需要展示每个插件所属的分类。
按目前的写法，一页 20 个插件要查 21 次库，希望合并成 2 次。

## 技术方案

### 1. 优化前（我们要改掉的写法）
逐行查分类：

```ts
const plugins = db.prepare('SELECT * FROM plugins LIMIT 20').all();
for (const p of plugins) {
  p.categories = db.prepare(
    'SELECT * FROM plugin_categories WHERE repository_id = ?'
  ).all(p.repository_id);
}
```

### 2. 优化后
改成一次批量查询：

```ts
const plugins = db.prepare('SELECT * FROM plugins LIMIT 20').all();
const ids = plugins.map(p => p.repository_id);
const cats = db.prepare(
  `SELECT * FROM plugin_categories WHERE repository_id IN (${ids.map(() => '?').join(',')})`
).all(...ids);
// 再按 repository_id 分组
```

## 预期
- 查询从 21 次降到 2 次
- 列表页显著变快
