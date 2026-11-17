# 方案：审计日志对外导出

## 背景
合规要求审计日志可导出留存。

## 技术方案

### 1. 导出接口
```ts
GET /api/admin/operations/export

const rows = db.prepare("SELECT * FROM operation_records").all();
return new Response(JSON.stringify(rows), {
  headers: { 'content-type': 'application/json' }
});
```

### 2. 定期归档
每天导出一次，打包发到对象存储。

## 预期
- 审计可留存
- 满足合规
