# 方案：基于访问日志的用户行为分析

## 背景
想了解用户在站内的行为路径，优化产品体验。
现有 server.mjs 每请求一行访问日志。

## 技术方案

### 1. 日志字段扩展
在业务层（route handler 里，能拿到当前用户上下文）补充用户标识：

```ts
// 在各 API route 的出口处补记一条
log.info('http_request', {
  request_id,
  method,
  route_path,
  status,
  duration_ms,
  user_id: ctx.userId,             // 新增，来自 currentContext()
  user_agent: req.headers['user-agent'],  // 新增
  referer: req.headers.referer,    // 新增
});
```

### 2. 行为路径还原
按 user_id 聚合，还原每个用户的页面访问序列。

### 3. 存储
日志写入 JSONL 文件，保留 90 天。

### 4. 分析
用脚本按 user_id 分组，统计路径频次。

## 预期
- 能看到用户完整的浏览路径
- 保留 90 天便于回溯
- 数据量大但都是文本，成本可控
