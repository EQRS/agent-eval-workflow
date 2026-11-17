# 方案：将 GitHub OAuth 登录改为支持多提供商

## 背景
目前只支持 GitHub 登录，用户希望能用 Gitee、GitLab 等登录。

## 技术方案

### 1. 数据模型改动
新建迁移 0043：

```sql
-- 新增身份绑定表
CREATE TABLE user_identities (
  user_id    TEXT NOT NULL REFERENCES users(id),
  provider   TEXT NOT NULL,
  provider_user_id TEXT NOT NULL,
  email      TEXT,
  created_at INTEGER NOT NULL,
  PRIMARY KEY (provider, provider_user_id)
);

-- 迁移历史数据
INSERT INTO user_identities (user_id, provider, provider_user_id, created_at)
SELECT id, 'github', github_id, created_at FROM users WHERE github_id IS NOT NULL;
```

### 2. 登录接口改动
现有 `/api/auth/github/start` 和 `/api/auth/github/callback` 改为通用：

```
GET /api/auth/{provider}/start
GET /api/auth/{provider}/callback
```

回调处理：
1. 从回调参数取 code
2. 调对应 provider 的 API 换 access_token
3. 用 access_token 拉用户信息
4. 查 user_identities 表找对应用户
5. 如果找不到，创建一个新用户

### 3. 会话保持
沿用现有的 session cookie 机制，无改动。

### 4. 数据库迁移
直接执行上述 SQL，现有 GitHub 用户的数据会自动迁移过来。

## 预期
- 支持 Gitee、GitLab 登录
- 现有用户无感知
- 一个用户可以绑定多个提供商
