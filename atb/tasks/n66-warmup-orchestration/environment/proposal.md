# 方案：服务启动时预热依赖

## 背景
服务冷启动时首屏慢，想在启动时预热：
数据库连接、配置、缓存。

## 技术方案

### 1. 启动钩子
```ts
// instrumentation.ts
export async function register() {
  // 预热数据库
  await db.prepare("SELECT 1").get();

  // 预热配置
  await loadSiteSettings();

  // 预热插件列表
  await listPlugins({});

  log.info("warmup_done");
}
```

### 2. 失败处理
预热失败就退出：

```ts
try {
  await warmup();
} catch (e) {
  log.error("warmup_failed", e);
  process.exit(1);
}
```

## 预期
- 首屏变快
- 启动时发现问题
