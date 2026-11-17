# 方案：新功能用 Feature Flag 灰度

## 背景
新功能上线有风险，希望用 feature flag 控制灰度。

## 技术方案

### 1. Flag 定义
```ts
export const FLAGS = {
  new_search:  0,   // 灰度比例 0-100
  new_editor:  0,
  new_comment: 0,
  new_profile: 0,
  new_upload:  0,
};
```

### 2. 灰度判定
按用户 ID 取模决定是否开启：
```ts
export function isEnabled(flag: keyof typeof FLAGS, userId: string): boolean {
  const percent = FLAGS[flag];
  if (percent <= 0) return false;
  if (percent >= 100) return true;
  return hashCode(userId) % 100 < percent;
}
```

### 3. 配置来源
从环境变量读取灰度比例，支持热更新。

### 4. 清理
功能全量后删除 flag。

## 预期
- 新功能可控灰度
- 出问题可快速关闭
