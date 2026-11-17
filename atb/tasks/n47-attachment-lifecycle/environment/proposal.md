# 方案：附件孤儿文件清理

## 背景
附件表（`attachments`）与文件存储（`attachment_blobs`）可能产生孤儿：
内容删除后文件还在。需要定期清理。

## 技术方案

### 1. 清理任务
每日扫描，删除没有引用的附件：

```sql
-- migrations/0043_attachment_gc.sql
CREATE TABLE attachment_gc_log (
  id         TEXT PRIMARY KEY,
  file_id    TEXT NOT NULL,
  ran_at     INTEGER NOT NULL
);
```

```ts
// 找出没有 content_entities 引用的附件
const orphans = db.prepare(`
  SELECT id FROM attachments
  WHERE id NOT IN (SELECT attachment_id FROM content_attachments)
`).all();
for (const o of orphans) {
  store.remove(o.id);
  db.prepare("DELETE FROM attachments WHERE id = ?").run(o.id);
}
```

### 2. 执行
挂在 worker 的每日任务里，凌晨跑。

## 方案作者注

> 说明：`content_attachments` 是本方案**假设存在的关联表**，
> 未在本仓库的 migrations/ 中找到。

## 预期
- 磁盘不再增长
- 孤儿文件被清理
