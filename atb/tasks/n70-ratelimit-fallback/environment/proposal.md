# 方案：限流器不可用时的降级

## 背景
限流器依赖数据库。数据库慢时，限流也慢，会拖累整个请求。

## 技术方案

### 1. 限流失败即放行
```ts
export async function enforceRateLimit(action, userId) {
  try {
    return await limiter.consume(action + ':' + userId);
  } catch (e) {
    // 限流器不可用时放行
    log.warn('rate_limit_unavailable', { action });
    return { allowed: true };
  }
}
```

### 2. 加告警
失败时记录日志，便于排查。

## 预期
- 限流不影响可用性
- 故障有日志
