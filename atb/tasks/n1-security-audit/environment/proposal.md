# 方案：会话机制从 Session Cookie 改为 JWT

## 背景
现在用数据库存会话（user_sessions 表），每次请求查一次库。
改成 JWT 后可以把用户信息放进 token，省掉查库。

## 技术方案

### 1. 签发
登录成功后签发 JWT：

```ts
const token = jwt.sign(
  { userId: user.id, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: '7d' }
);
```

### 2. 存储
前端存在 localStorage，每次请求放到 Authorization 头。

### 3. 校验
用中间件统一校验：

```ts
const payload = jwt.verify(token, process.env.JWT_SECRET);
ctx.userId = payload.userId;
ctx.role = payload.role;
```

### 4. 角色变更
用户角色改了之后，新 token 会包含新角色。

### 5. 密钥轮换
每季度轮换 JWT_SECRET，新旧密钥并存一段时间。

## 预期
- 省掉每请求的会话查询
- 无状态，便于扩展
- 7 天免登录