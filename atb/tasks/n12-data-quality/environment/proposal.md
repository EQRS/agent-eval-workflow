# 方案：数据校验规则集中化

## 背景
现在各处都在做手动校验，代码重复。集中到一处。

## 技术方案

### 1. 统一校验函数
```ts
export function validatePlugin(input: unknown): Plugin {
  if (!input.display_name || input.display_name.length > 100) throw err.input('名称非法');
  if (!input.description || input.description.length > 5000) throw err.input('描述非法');
  if (!['active','archived'].includes(input.status)) throw err.input('状态非法');
  return input as Plugin;
}
```

### 2. 调用点
所有写插件的地方统一调用：
- 用户提交接口
- 管理员后台接口
- 批量导入
- 数据修复脚本

### 3. 移除旧校验
删除各处的重复校验代码。

## 预期
- 代码更简洁（DRY）
- 校验规则一处维护
