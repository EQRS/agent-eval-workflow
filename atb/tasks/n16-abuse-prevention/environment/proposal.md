# 方案：GitHub 登录建号的防刷改造

## 背景
有人用脚本批量注册 GitHub 账号，然后批量发表评测和评论。
需要在建号环节加防护。

## 技术方案

### 1. 频率限制
在 GitHub OAuth 回调里限制建号频率：

```ts
const count = db.prepare(
  "SELECT COUNT(*) AS c FROM users WHERE created_at > ?"
).get(Date.now() - 3600_000).c;
if (count > 10) throw err.rateLimited('注册过于频繁');
```

### 2. 图形验证码
同一来源超过 3 次后要求输入验证码。

### 3. 手机验证
新用户需绑定手机号才能发表内容。

## 预期
- 有效阻止批量注册
- 正常用户几乎无感知
