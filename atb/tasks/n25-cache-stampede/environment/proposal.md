# 方案：站点启动时预热热点数据

## 背景
站点重启后第一批访问很慢，因为什么缓存都没有。想在启动时预热。

## 技术方案

### 1. 预热内容
启动时把插件列表查出来放内存：

```ts
const hotPlugins = db.prepare(
  "SELECT * FROM plugins ORDER BY updated_at DESC LIMIT 100"
).all();
global.__hotPlugins = hotPlugins;
```

### 2. 访问时
先查内存，没有再查库。

### 3. 失效
每小时重新加载一次。

## 预期
- 首屏变快
- 数据库压力下降
