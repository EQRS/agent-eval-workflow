# 方案：用户数据敏感度分级

## 背景
库里混存着公开的插件信息与用户隐私数据。
需要给字段打上敏感度标签，便于统一管控。

## 技术方案

### 1. 分级定义
```ts
export const SENSITIVITY = {
  public:    1,   // 公开信息
  internal:  2,   // 内部可见
  sensitive: 3,   // 敏感个人信息
  critical:  4,   // 关键数据
} as const;
```

### 2. 字段标注
用注释标注每张表的字段等级：

```sql
-- sensitivity: public
CREATE TABLE plugins (
  id TEXT PRIMARY KEY,
  ...
);
```

### 3. 统一管控
所有 sensitive 及以上的字段：
- 日志里不出现
- 导出时脱敏
- 备份时加密

## 预期
- 数据敏感情形清晰
- 管控有据可依
