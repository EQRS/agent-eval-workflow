# 方案：插件归属争议的申诉流程

## 现状
插件归属（`plugin_ownerships`）目前由系统自动匹配 GitHub 账号或人工认领确定。
出现争议时需要一套申诉流程。

## 技术方案

### 1. 争议表
```sql
-- migrations/0043_ownership_disputes.sql
CREATE TABLE ownership_disputes (
  id            TEXT PRIMARY KEY,
  plugin_id     TEXT NOT NULL REFERENCES plugins(id),
  claimant_id   TEXT NOT NULL REFERENCES users(id),
  evidence      TEXT,
  status        TEXT NOT NULL DEFAULT 'pending',
  created_at    INTEGER NOT NULL
);
```

### 2. 申诉
任何用户都可以对任意插件提出归属争议，填写证据。

### 3. 处理
管理员查看证据后，直接改 `plugin_ownerships` 的归属人。

## 预期
- 争议有据可查
- 归属可纠正
