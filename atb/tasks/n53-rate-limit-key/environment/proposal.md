# 方案：限流按 IP 维度补充

## 背景
现在的限流是按用户（`<action>:<userId>`）。
匿名用户没有 userId，无法限流。需要补充 IP 维度。

## 技术方案

### 1. 键的构成
```ts
function rateLimitKey(action: string, ctx: CallerContext, req: Request) {
  const subject = ctx.userId ?? req.headers.get('x-forwarded-for') ?? 'unknown';
  return `${action}:${subject}`;
}
```

### 2. 应用到匿名接口
浏览、搜索等匿名接口都加上限流。

### 3. 额度
匿名用户比登录用户更严格。

## 预期
- 匿名请求也能限流
- 防止爬虫
