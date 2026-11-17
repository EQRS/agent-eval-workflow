# 方案：服务优雅停机改造

## 背景
部署重启时会中断正在处理的请求，需要优雅停机。

## 技术方案

### 1. 监听信号
```ts
process.on('SIGTERM', async () => {
  server.close();
  await sleep(5000);
  process.exit(0);
});
```

### 2. 等待
给 5 秒让进行中的请求完成。

### 3. 部署配置
compose 里设 `stop_grace_period: 15s`。

## 预期
- 重启不丢请求
- 部署更平滑
