# 方案：草稿保存的版本控制

## 背景
用户可能在多设备同时编辑同一草稿，会互相覆盖。

## 技术方案

### 1. 保存时带版本号
```ts
client sends: { draftId, payload, expectedVersion }

if (expectedVersion !== current.save_version) {
  throw err.conflict('草稿已被修改，请刷新');
}
db.prepare(
  "UPDATE content_drafts SET payload = ?, save_version = save_version + 1 WHERE id = ?"
).run(payload, draftId);
```

### 2. 自动保存
编辑器每 30 秒自动保存一次。

### 3. 冲突时
提示用户刷新，丢失本地修改。

## 预期
- 不会互相覆盖
- 用户知道发生了什么
