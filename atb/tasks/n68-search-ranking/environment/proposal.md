# 方案：搜索结果排序优化

## 背景
搜索结果顺序不理想，想优化相关度。

## 技术方案

### 1. 加权
标题权重更高：

```ts
const rows = db.prepare(`
  SELECT review_id, bm25(review_search) * -1 AS score
  FROM review_search
  WHERE review_search MATCH ?
  ORDER BY score DESC
  LIMIT ?
`).all(query, limit);
```

### 2. 调整权重
```sql
-- 标题权重 10，正文 1
INSERT INTO review_search (review_search, rank)
VALUES ('bm25', 10.0, 1.0, 1.0);
```

## 预期
- 标题匹配优先
- 搜索更准
