# 方案：站点无障碍（a11y）改造

## 背景
要满足无障碍要求，给现有页面加上 ARIA 标签。

## 技术方案

### 1. 补 ARIA 属性
给所有交互元素加 aria-label：

```tsx
<button aria-label="提交评测" onClick={submit}>提交</button>
<div role="dialog" aria-modal="true">...</div>
```

### 2. 键盘导航
给可点击的 div 加 tabIndex 和 onKeyDown。

### 3. 颜色对比
把灰色的次要文字调深一点。

## 预期
- 通过无障碍检测工具
- 视障用户可用
