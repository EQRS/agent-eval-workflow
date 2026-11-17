# 方案：管理员批量导入用户

## 背景
管理员需要从 CSV 批量导入用户。

## 技术方案

### 1. 上传 CSV
```ts
const csv = await req.text();
const rows = csv.split('\n').map(line => line.split(','));

for (const [githubLogin, role] of rows) {
  db.prepare(
    "INSERT INTO users (id, nickname, role) VALUES (?, ?, ?)"
  ).run(newId(), githubLogin, role);
}
```

### 2. 结果
返回成功与失败数量。

## 预期
- 一次导入多个用户
- 管理员省事
