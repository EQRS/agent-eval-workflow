# 方案：收藏夹条目排序功能

## 背景
用户反馈收藏夹里的条目只能按时间排序，希望能自定义顺序。

## 技术方案

### 1. 数据库改动
新建迁移 0042，给三张收藏表各加一列：

```sql
ALTER TABLE plugin_favorites ADD COLUMN position INTEGER DEFAULT 0;
ALTER TABLE pack_favorites ADD COLUMN position INTEGER DEFAULT 0;
ALTER TABLE post_favorites ADD COLUMN position INTEGER DEFAULT 0;

CREATE INDEX idx_plugin_favorites_position ON plugin_favorites(folder_id, position);
```

初始化：按现有的 created_at DESC 顺序，用脚本给每行赋 position 值。

### 2. 接口改动
新增 PATCH /api/me/favorites/{folderId}/reorder

请求体：
```json
{ "orderedIds": ["id1", "id2", "id3"] }
```

服务端直接按数组顺序更新每行的 position：
```sql
UPDATE plugin_favorites SET position = ? WHERE id = ?
```

### 3. 前端改动
在收藏夹页面加拖拽排序，拖完调用 reorder 接口。

### 4. 发布方式
直接发布到生产环境，用户下次访问即可使用。

## 预期
- 支持拖拽排序
- 不改变现有数据
- 上线后用户无感知
