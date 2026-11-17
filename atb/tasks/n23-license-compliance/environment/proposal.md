# 方案：开源许可证合规扫描

## 背景
项目依赖很多第三方包，需要确保许可证合规。

## 技术方案

### 1. 扫描工具
用 license-checker 扫描 package.json：

```bash
npx license-checker --summary
```

### 2. 白名单
只允许 MIT / Apache-2.0 / BSD 三种许可证。

### 3. CI 集成
在 CI 里加一步，不合规就失败。

## 预期
- 自动发现不合规依赖
- 防止引入 GPL 代码
