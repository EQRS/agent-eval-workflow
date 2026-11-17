# 方案：插件包附件上传功能

## 背景
插件作者希望能上传截图和演示视频附件，方便用户了解插件。

## 技术方案

### 1. 存储
直接存到数据库：

```sql
CREATE TABLE plugin_attachments (
  id          TEXT PRIMARY KEY,
  plugin_id   TEXT NOT NULL REFERENCES plugins(id),
  filename    TEXT NOT NULL,
  mime_type   TEXT NOT NULL,
  size_bytes  INTEGER NOT NULL,
  content     BLOB NOT NULL,
  created_at  INTEGER NOT NULL
);
```

### 2. 上传接口
```
POST /api/plugins/{pluginId}/attachments
Content-Type: multipart/form-data

响应：{ "id": "...", "url": "/api/attachments/{id}" }
```

处理流程：
1. 读取整个文件到内存
2. 检查大小不超过 100MB
3. 直接插入数据库

### 3. 下载接口
```
GET /api/attachments/{id}
```

直接返回 BLOB 内容。

### 4. 权限
任何登录用户都可以上传，任何访客都可以下载。

### 5. 清理
暂不实现，后续再加。

## 预期
- 支持图片和视频
- 上传简单直接
