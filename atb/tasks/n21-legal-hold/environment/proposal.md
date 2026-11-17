# 方案：用户注销后的数据清理

## 背景
用户申请注销账号，需要在 30 天内清理其所有数据。

## 技术方案

### 1. 立即清理
注销请求通过后，立即删除所有业务数据：

```sql
DELETE FROM reviews WHERE author_id = ?;
DELETE FROM comments WHERE author_id = ?;
DELETE FROM review_revisions WHERE review_id IN (...);
DELETE FROM plugin_favorites WHERE user_id = ?;
DELETE FROM user_sessions WHERE user_id = ?;
DELETE FROM operation_records WHERE actor_user_id = ?;
DELETE FROM users WHERE id = ?;
```

### 2. 保留
只保留审计记录（operation_records 的 system 身份部分）。

### 3. 期限
30 天内完成，超过则人工介入。

## 预期
- 用户注销后数据彻底清除
- 符合删除权要求
