# 方案：统一错误处理重构

## 背景
现在各处抛错方式不统一，有的抛 Error，有的抛字符串。要统一。

## 技术方案

### 1. 统一错误类
```ts
export class HttpError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}
```

### 2. 业务代码
所有抛错改用 HttpError：

```ts
if (!user) throw new HttpError(404, '用户不存在');
if (user.role !== 'admin') throw new HttpError(403, '无权操作');
```

### 3. 全局捕获
在 API 层统一 catch 并返回：

```ts
try {
  return await handler(req);
} catch (e) {
  if (e instanceof HttpError) {
    return NextResponse.json({ error: e.message }, { status: e.status });
  }
  return NextResponse.json({ error: '服务器错误' }, { status: 500 });
}
```

## 预期
- 错误处理统一
- 代码更清晰
