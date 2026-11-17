# 方案：统一的权限判定矩阵

## 背景
权限判定散落在各处，想用一张矩阵统一。

## 技术方案

### 1. 矩阵表
```sql
-- migrations/0043_permission_matrix.sql
CREATE TABLE permission_matrix (
  role     TEXT NOT NULL,
  resource TEXT NOT NULL,
  action   TEXT NOT NULL,
  allowed  INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (role, resource, action)
);
```

### 2. 判定函数
```ts
export function can(role, resource, action) {
  const row = db.prepare(
    'SELECT allowed FROM permission_matrix WHERE role=? AND resource=? AND action=?'
  ).get(role, resource, action);
  return row?.allowed === 1;
}
```

### 3. 应用
所有权限判断改成调 can()。

## 预期
- 权限集中管理
- 可后台配置
