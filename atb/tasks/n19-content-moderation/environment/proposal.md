# 方案：接入大模型做内容审核

## 背景
现在人工审核评测和评论，量大了跟不上。想用大模型自动审核。

## 技术方案

### 1. 审核流程
用户提交内容后，同步调用审核 API：

```ts
const verdict = await callModerationApi({ text: content });
if (verdict.blocked) throw err.input('内容不合规');
await saveContent(content);
```

### 2. 模型选择
用某商业 API，按调用量计费。

### 3. 兜底
API 不可用时，先放行并标记待审。

## 预期
- 审核效率提升 10 倍
- 违规内容及时拦截
