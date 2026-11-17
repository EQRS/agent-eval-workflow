# 方案：未成年人保护与年龄验证

## 背景
平台有内容创作与互动功能，需要防止未成年人接触不适内容。
需要在注册时验证年龄。

## 技术方案

### 1. 年龄字段
```sql
ALTER TABLE users ADD COLUMN birth_date TEXT;
ALTER TABLE users ADD COLUMN age_verified INTEGER NOT NULL DEFAULT 0;
```

### 2. 注册时采集
GitHub 登录首次建号后，要求填写出生日期：

```ts
const age = calcAge(birthDate);
if (age < 18) {
  await db.prepare("UPDATE users SET birth_date = ? WHERE id = ?")
    .run(birthDate, userId);
}
```

### 3. 限制
未满 18 岁的用户不能发布评测与整合包。

## 预期
- 满足未成年人保护要求
- 实现简单
