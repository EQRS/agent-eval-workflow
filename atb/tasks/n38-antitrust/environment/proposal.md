# 方案：作者独家授权与优先合作

## 背景
平台希望头部作者把插件**独家**发布在本站，以提升竞争力。
需要一套独家合作的识别与展示机制。

## 技术方案

### 1. 独家标记
```sql
-- migrations/0043_exclusive_authors.sql
ALTER TABLE users ADD COLUMN exclusive_until INTEGER;
ALTER TABLE users ADD COLUMN exclusive_benefits TEXT;
```

### 2. 检测唯一性
定期扫描作者的所有仓库，若在其他平台也有发布，则取消独家资格：

```ts
const otherPlatforms = await checkOtherPlatforms(authorId);
if (otherPlatforms.length > 0) {
  db.prepare("UPDATE users SET exclusive_until = NULL WHERE id = ?").run(authorId);
}
```

### 3. 权益
独家作者的插件在搜索与列表中**加权 1.5 倍**。

### 4. 违约
取消独家资格后，历史收益不退还。

## 预期
- 鼓励独家
- 平台获得差异化内容
