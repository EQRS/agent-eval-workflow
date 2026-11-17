# 方案：评测正文的冷热分层存储

## 背景
数据库文件持续膨胀（当前约 1.2MB，随评测增长）。
评测正文（body_markdown）是主要占用，但 6 个月前的极少被读。

## 技术方案

### 1. 压缩副本表
```sql
CREATE TABLE review_revisions_cold (
  revision_id   TEXT PRIMARY KEY,
  body_gzip     BLOB NOT NULL,
  original_size INTEGER NOT NULL,
  archived_at   INTEGER NOT NULL
);
CREATE INDEX idx_rrc_archived ON review_revisions_cold(archived_at);
```

### 2. 归档任务
定期把 6 个月前的正文压缩写入冷表：

```ts
const rows = db.prepare(
  "SELECT id, body_markdown FROM review_revisions WHERE submitted_at < ?"
).all(cutoff);
for (const r of rows) {
  const gz = gzipSync(Buffer.from(r.body_markdown, 'utf8'));
  db.prepare(
    "INSERT OR REPLACE INTO review_revisions_cold (revision_id, body_gzip, original_size, archived_at) VALUES (?, ?, ?, ?)"
  ).run(r.id, gz, r.body_markdown.length, nowMs());
}
```

### 3. 读取回退
读取详情时，若热表正文为空则查冷表：

```ts
function readBody(db, revisionId) {
  const hot = db.prepare("SELECT body_markdown FROM review_revisions WHERE id = ?").get(revisionId);
  if (hot?.body_markdown) return hot.body_markdown;
  const cold = db.prepare("SELECT body_gzip FROM review_revisions_cold WHERE revision_id = ?").get(revisionId);
  return cold ? gunzipSync(cold.body_gzip).toString('utf8') : '';
}
```

### 4. 清理热表
归档完成后清空热表正文以释放空间：
```sql
UPDATE review_revisions SET body_markdown = '' WHERE submitted_at < :cutoff;
```

## 预期
- 主表体积减少 70%
- 冷数据读取延迟可接受
- 可随时从冷表恢复
