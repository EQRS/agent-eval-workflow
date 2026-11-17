# 方案：统一写接口权限校验

## 背景
各写接口的权限校验散落，容易漏。想统一。

## 技术方案

### 1. 中间件
```ts
export function requireAuth(handler) {
  return async (req) => {
    const session = await getSession(req);
    if (!session) return unauthorized();
    return handler(req, session);
  };
}
```

### 2. 包一层
所有写接口用这个包：

```ts
export const POST = requireAuth(async (req, session) => {
  // ...
});
```

## 预期
- 权限校验统一
- 不会漏
