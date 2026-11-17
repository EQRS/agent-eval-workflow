# ATB 题库索引

> 结构借鉴 Harbor Index（80 题 benchmark）
> **当前：40 题**（8 M 系列 + 32 N 系列），覆盖 **24 个领域**
> 更新：2026-11-17
> **状态：40 题全部实跑（每题 1 S 组 + 3 T 组）**
>
> ⚠️ **已知前提错误 6 题（未修，如实记录）**：N-3、N-12、N-15、N-28、N-30、N-31
> —— 这些题的素材引用了不存在的对象，T 组产出有一半在「证明不存在」。
>
> ✅ **优质考点**：N-29/N-30/N-32（无用功类）、N-24（域外知识）、N-27（SSRF）、N-21（删除链）

---

## 目录结构

```
tasks/
  └── <领域>-<题目名>/
      ├── instruction.md       # 任务说明
      ├── environment/
      │   └── proposal.md      # 素材（待评审的方案）
      ├── solution/
      ├── tests/
      │   └── criteria.md      # 判据清单（agent 不可见）
      └── task.toml            # 元数据
```

---

## 全部 40 题

| # | 题目目录 | 领域 | 难度 | 主题 |
|---|---|---|---|---|
| 1 | `m10-idempotent-queue` | consistency | L2 | 发现任务幂等去重方案 |
| 2 | `m11-fulltext-search` | search | L2 | 插件全文搜索方案 |
| 3 | `m12-schedule-timing` | scheduling | L3 | 定时任务调度时间改造方案 |
| 4 | `m13-data-export` | compliance | L3 | 用户数据导出方案 |
| 5 | `m6-sorting-permission` | database | L2 | 收藏夹排序功能方案 |
| 6 | `m7-multi-provider-login` | security | L2 | 多提供商登录方案 |
| 7 | `m8-file-upload` | storage | L2 | 插件附件上传方案 |
| 8 | `m9-scoring-algorithm` | algorithm | L3 | 插件评分加权排序方案 |
| 9 | `n1-security-audit` | security | L3 | JWT 会话改造方案 |
| 10 | `n10-observability` | observability | L2 | 全链路追踪接入方案 |
| 11 | `n11-supply-chain` | supply-chain | L2 | 依赖自动升级方案 |
| 12 | `n12-data-quality` | data-quality | L2 | 数据校验规则集中化方案 |
| 13 | `n13-product-decision` | product | L2 | 功能开关灰度方案 |
| 14 | `n14-deployment` | deployment | L3 | 零停机数据迁移方案 |
| 15 | `n15-cost-optimization` | cost | L2 | 存储成本优化方案 |
| 16 | `n16-abuse-prevention` | security | L2 | 用户注册的防刷方案 |
| 17 | `n17-data-leak` | privacy | L3 | 日志采集的用户行为分析方案 |
| 18 | `n18-rate-bypass` | security | L3 | API 限流的分布式改造方案 |
| 19 | `n19-content-moderation` | security | L3 | AI 内容审核接入方案 |
| 20 | `n2-distributed-consistency` | consistency | L3 | 跨实例缓存失效方案 |
| 21 | `n20-payment-security` | security | L3 | 插件付费下载的订单方案 |
| 22 | `n21-legal-hold` | compliance | L3 | 用户注销数据清理方案 |
| 23 | `n22-accessibility` | compliance | L2 | 无障碍改造方案 |
| 24 | `n23-license-compliance` | compliance | L2 | 开源许可证扫描方案 |
| 25 | `n24-data-residency` | compliance | L3 | 数据跨境存储方案 |
| 26 | `n25-cache-stampede` | performance | L2 | 热点数据的预热方案 |
| 27 | `n26-idempotent-api` | consistency | L3 | 写接口的幂等改造方案 |
| 28 | `n27-webhook-delivery` | integration | L3 | Webhook 推送方案 |
| 29 | `n28-file-sync` | storage | L3 | 插件包多地域同步方案 |
| 30 | `n29-graceful-shutdown` | operations | L2 | 服务优雅停机方案 |
| 31 | `n3-migration-evolution` | migration | L3 | 用户表拆分方案 |
| 32 | `n30-n-plus-one` | performance | L2 | 列表页查询优化方案 |
| 33 | `n31-transaction-boundary` | consistency | L3 | 跨模块事务改写方案 |
| 34 | `n32-error-handling` | code-quality | L2 | 统一错误处理重构方案 |
| 35 | `n4-capacity-planning` | performance | L2 | 数据库连接池改造方案 |
| 36 | `n5-incident-rca` | observability | L3 | 线上 500 错误排查方案 |
| 37 | `n6-code-review` | code-review | L2 | 批量重命名的接口重构方案 |
| 38 | `n7-test-design` | testing | L2 | 端到端测试覆盖提升方案 |
| 39 | `n8-api-contract` | api-design | L3 | API 版本化改造方案 |
| 40 | `n9-concurrency` | concurrency | L3 | 库存扣减的乐观锁方案 |

---

## 领域分布（20 个）

| 领域 | 题数 |
|---|---|
| security | 6 |
| compliance | 5 |
| consistency | 4 |
| performance | 3 |
| storage | 2 |
| observability | 2 |
| search | 1 |
| scheduling | 1 |
| database | 1 |
| algorithm | 1 |
| supply-chain | 1 |
| data-quality | 1 |
| product | 1 |
| deployment | 1 |
| cost | 1 |
| privacy | 1 |
| integration | 1 |
| operations | 1 |
| migration | 1 |
| code-quality | 1 |
| code-review | 1 |
| testing | 1 |
| api-design | 1 |
| concurrency | 1 |

---

## 前提核实（出题纪律第 6 条）

**每题出题前必须通过**：

```bash
node scripts/verify-task-prereqs.mjs
```

**检查内容**：

| # | 检查 |
|---|---|
| 1 | 素材提到的表名 → migrations/ 里必须真的存在 |
| 2 | 素材要新建的表 → 素材里必须有 CREATE TABLE |
| 3 | 素材提到的 .ts 文件路径 → 必须存在 |
| 4 | 已存在的表 → 报告它的外键数与触发器数 |

**当前状态**：✅ **检查 40 题，异常 0 题**

---

## 六条出题纪律

| # | 纪律 | 违反后果 |
|---|---|---|
| 1 | **S 组绝不能被提示维度** | 测不出盲区 |
| 2 | **判据清单绝不能让 agent 读到** | M-9v1 泄题 |
| 3 | **隐患必须需要专业视角** | 半合格 |
| 4 | **每题用全新 subagent** | 经验污染 |
| 5 | **素材要看起来正确** | 一读代码就发现 |
| **6** | **方案提到的每个表/列/接口，必须先核实存在** | **N-3/N-12/N-15 前提错** |

---

## 隐患隐藏度（出题优先级）

| 类型 | 隐藏度 | 例子 |
|---|---|---|
| 代码内信息 | 低 | payload 丢字段 |
| 跨系统交互 | 中 | 03:00 撞备份 timer |
| 对抗性思维 | 中高 | 评分被刷、SSRF |
| **域外知识** | **最高** | **PIPL vs GDPR** |

> **域外知识 > 对抗性思维 > 跨系统交互 > 代码内信息**

---

## 运行方式

**S 组**：1 个 subagent，只给任务，**不提示维度**

**T 组**：3 个 subagent，各带专业身份（安全/性能/运维）

**判据**：按 `tests/criteria.md` 逐项打勾

**并发上限**：18（超了会报错，需分批）
