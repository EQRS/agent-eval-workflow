# 方案：表单防重复提交

## 背景
用户双击提交按钮会重复创建。需要防重复提交。

## 技术方案

### 1. 前端禁用按钮
提交时禁用按钮，防止重复点击：

```tsx
const [submitting, setSubmitting] = useState(false);
<button disabled={submitting} onClick={async () => {
  setSubmitting(true);
  await submit();
  setSubmitting(false);
}}>提交</button>
```

### 2. 后端加锁
用内存 Set 记录正在处理的请求：

```ts
const inFlight = new Set<string>();

if (inFlight.has(userId)) throw err.conflict('请勿重复提交');
inFlight.add(userId);
try { await handle(); } finally { inFlight.delete(userId); }
```

## 预期
- 不会重复提交
