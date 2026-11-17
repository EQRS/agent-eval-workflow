# 方案：列表分页改成游标模式

## 背景
深翻页（OFFSET 很大）越来越慢，想改成游标分页。

## 技术方案

### 1. 游标格式
```ts
// 用 updated_at 作为游标
const cursor = lastItem.updated_at;

const rows = db.prepare(`
  SELECT * FROM plugins
  WHERE updated_at < ?
  ORDER BY updated_at DESC
  LIMIT 20
`).all(cursor);
```

### 2. 返回
```ts
return { items: rows, nextCursor: rows.at(-1)?.updated_at };
```

## 预期
- 深翻页不变慢
- 用户无感知
