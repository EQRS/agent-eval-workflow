# 方案：引入数据库连接池提升并发

## 背景
现在 better-sqlite3 是单连接，所有请求排队。引入连接池提升并发。

## 技术方案

### 1. 连接池配置
```ts
const pool = createPool({
  connectionString: process.env.DATABASE_URL,
  max: 20,              // 最大连接数
  min: 5,               // 最小保持
  idleTimeoutMillis: 30000,
});
```

### 2. 使用
```ts
const conn = await pool.acquire();
const rows = conn.prepare('SELECT * FROM plugins').all();
await pool.release(conn);
```

### 3. 池大小计算
按业界公式：连接数 = CPU 核心数 × 2 + 磁盘数。
4 核 1 盘 = 9，取 20 留余量。

### 4. 监控
暴露池状态到 /api/health。

## 预期
- 并发请求不再排队
- 吞吐提升 3-5 倍