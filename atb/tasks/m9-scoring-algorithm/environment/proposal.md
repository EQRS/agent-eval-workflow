# 方案：插件评分的加权排序算法

## 背景
当前插件列表按"最新收录"排序，用户抱怨找不到优质插件。
产品希望改成"按质量排序"，让好评多的插件排在前面。

## 技术方案

### 1. 评分公式
采用业界通用的贝叶斯平均（Bayesian Average），避免新插件因样本少而排名过低：

```
score = (v / (v + m)) * R + (m / (v + m)) * C
```

- v = 该插件的评测数
- R = 该插件的平均分（1-5）
- m = 调节参数，取 10（业界常用值）
- C = 全站平均分

### 2. 数据来源
从 reviews 表聚合：

```sql
SELECT plugin_id,
       COUNT(*) AS v,
       AVG(rating) AS R
FROM reviews
WHERE rating IS NOT NULL
GROUP BY plugin_id
```

### 3. 实现
- 每小时跑一次定时任务，重算所有插件的 score，写入 plugins.score 列
- 列表查询按 score DESC 排序
- 新建迁移加 plugins.score 列（REAL，默认 0）

### 4. 参数 m 的调整
初期 m=10，后续根据数据分布调整。

## 预期
- 优质插件自然浮上来
- 新插件有公平曝光机会
- 排序算法业界通用，风险低
