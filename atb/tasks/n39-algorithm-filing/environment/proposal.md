# 方案：插件推荐算法上线

## 背景
想给用户推荐可能感兴趣的插件，提升发现效率。
用简单的协同过滤实现。

## 技术方案

### 1. 用户行为采集
```sql
-- migrations/0043_user_behavior.sql
CREATE TABLE user_behaviors (
  id         TEXT PRIMARY KEY,
  user_id    TEXT NOT NULL REFERENCES users(id),
  plugin_id  TEXT NOT NULL REFERENCES plugins(id),
  action     TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
```

### 2. 推荐计算
每日凌晨跑一次：给每个用户推荐相似用户喜欢的插件。

```ts
const recs = computeRecommendations();
await db.prepare("INSERT INTO recommendations ...").run(recs);
```

### 3. 推荐结果表
```sql
-- migrations/0044_recommendations.sql
CREATE TABLE recommendations (
  id         TEXT PRIMARY KEY,
  user_id    TEXT NOT NULL REFERENCES users(id),
  plugin_id  TEXT NOT NULL REFERENCES plugins(id),
  score      REAL NOT NULL,
  created_at INTEGER NOT NULL
);
```

### 4. 展示
首页增加「为你推荐」区块。

## 预期
- 提升插件发现率
- 用户留存提升
