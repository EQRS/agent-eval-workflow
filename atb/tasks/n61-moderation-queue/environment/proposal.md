# 方案：新增「审核员」角色并分配待审任务

## 背景
待审内容（`content_submissions` 的 `manual_review` 状态）积压，
目前只有管理员能处理，需要新增「审核员」角色分担。

## 技术方案

### 1. 扩展角色闭集
```sql
-- migrations/0043_moderator_role.sql
ALTER TABLE users DROP CONSTRAINT users_role_check;
ALTER TABLE users ADD CONSTRAINT users_role_check
  CHECK (role IN ('user', 'admin', 'moderator'));
```

### 2. 分配表
```sql
CREATE TABLE review_assignments (
  id            TEXT PRIMARY KEY,
  submission_id TEXT NOT NULL REFERENCES content_submissions(id),
  reviewer_id   TEXT NOT NULL REFERENCES users(id),
  assigned_at   INTEGER NOT NULL
);
```

### 3. 分配逻辑
每 5 分钟扫描待审队列，轮询分配给审核员：

```ts
const pending = db.prepare(
  "SELECT * FROM content_submissions WHERE processing_state = 'manual_review' LIMIT 50"
).all();
const reviewers = db.prepare(
  "SELECT id FROM users WHERE role = 'moderator' AND account_status = 'active'"
).all();
pending.forEach((p, i) => {
  db.prepare(
    "INSERT INTO review_assignments (id, submission_id, reviewer_id, assigned_at) VALUES (?,?,?,?)"
  ).run(newId(), p.id, reviewers[i % reviewers.length].id, Date.now());
});
```

### 4. 审核入口
审核员通过 `/admin/submissions` 页面处理被分配的内容。

## 预期
- 审核压力分散
- 减少管理员负担
