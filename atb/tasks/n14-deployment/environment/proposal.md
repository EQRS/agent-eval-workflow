# 方案：插件表零停机迁移

## 背景
需要给 plugins 表加分类字段，且要改主键结构。不能停机。

## 技术方案

### 1. 新建 plugins_v2

```sql
CREATE TABLE plugins_v2 (
  id                TEXT    PRIMARY KEY,
  repository_id     TEXT    NOT NULL REFERENCES repositories(id) ON UPDATE RESTRICT ON DELETE RESTRICT,
  plugin_path       TEXT    NOT NULL CHECK (trim(plugin_path) <> ''),
  created_at        INTEGER NOT NULL CHECK (created_at >= 0),
  updated_at        INTEGER NOT NULL CHECK (updated_at >= 0),
  display_name      TEXT,
  description       TEXT,
  author_login      TEXT,
  manifest_version  TEXT,
  info_updated_at   INTEGER CHECK (info_updated_at IS NULL OR info_updated_at >= 0),
  UNIQUE (repository_id, plugin_path COLLATE BINARY)
);
```

### 2. 双写
```ts
// 新旧两张表同时写
await db.prepare('INSERT INTO plugins_v2 (...) VALUES (...)').run(...);
await db.prepare('INSERT INTO plugins (...) VALUES (...)').run(...);
```

### 3. 迁移历史数据
```sql
INSERT INTO plugins_v2 SELECT * FROM plugins;
```

### 4. 切换读
代码改成读 plugins_v2，部署上线。

### 5. 清理
确认无误后删除 plugins 表。

## 预期
- 不停机
- 数据不丢
- 出问题可回滚