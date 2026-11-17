# 方案：插件作者作为「平台内经营者」的管理

## 背景
付费插件上线后，作者在平台上售卖。需要明确平台与作者的责任边界。

## 技术方案

### 1. 作者资质表
```sql
-- migrations/0043_merchant_info.sql
CREATE TABLE merchant_info (
  user_id      TEXT PRIMARY KEY REFERENCES users(id),
  real_name    TEXT NOT NULL,
  id_number    TEXT NOT NULL,
  license_no   TEXT,
  verified_at  INTEGER
);
```

### 2. 入驻流程
作者首次售卖前填写实名信息，平台人工核验后置 verified_at。

### 3. 责任声明
在插件详情页展示：「本插件由作者提供，平台仅提供技术服务。」

## 预期
- 责任边界清晰
- 交易可追溯
