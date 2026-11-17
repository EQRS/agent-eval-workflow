# 方案：users 表拆分为 users + user_profiles

## 背景
现在 users 表混了身份与资料两类字段，查询时经常把大字段一起带出来。
拆成两张表，让身份查询只走 users。

## 技术方案

### 1. 新表
```sql
CREATE TABLE user_profiles (
  user_id     TEXT PRIMARY KEY REFERENCES users(id),
  nickname    TEXT NOT NULL,
  avatar_url  TEXT,
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);
```

### 2. 数据迁移
```sql
INSERT INTO user_profiles (user_id, nickname, avatar_url, created_at, updated_at)
SELECT id, nickname, avatar_url, created_at, updated_at FROM users;
```

### 3. 从 users 删除
```sql
ALTER TABLE users DROP COLUMN nickname;
ALTER TABLE users DROP COLUMN avatar_url;
```

### 4. 双写
迁移完成前，代码同时写两张表。

### 5. 回滚
如果出问题，把数据从 user_profiles 拷回 users。

## 预期
- users 表变窄，身份查询变快
- 迁移过程透明
