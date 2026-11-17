# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **项目已有重试机制，方案重复造轮子**
  `src/server/modules/tasks/constants.ts` 有
  `RETRY_BACKOFF_MS = [30s, 120s]` 与 `MAX_RECOVERABLE_ATTEMPTS = 3`。
  `src/server/adapters/catalog/github-discovery.ts` 也有自己的重试与预算控制。

- [ ] **没有区分「可重试」与「不可重试」错误**
  §1 的 catch 捕获**所有错误**都重试。
  若 GitHub 返回 404（仓库不存在）或 401（token 失效），
  重试 3 次**毫无意义**，反而延长失败时间。
  项目 `worker/handlers.ts` 有 terminal() 与 error() 的区分。

### L4 系统性
- [ ] **重试期间没有幂等保护，会产生重复副作用**
  §2 的 `UPDATE plugins SET description` 本身是幂等的（同样的值）。
  但若换成 `INSERT`（如新建插件记录），重试会**插入两条**。
  项目用 `operation_records` 的 `UNIQUE` 做幂等，方案没有。

- [ ] **重试没有总时长上限，会拖垮 worker**
  §1 的 `times = 3` 与退避 1+2+4 = 7 秒。
  但若 `fn` 本身超时 30 秒，总时长 = 3 × 30 + 7 = **97 秒**。
  `TASK_DEADLINE_MS = 300s` 内还行，
  但若多个任务并发，会**拖垮 worker**。

- [ ] **没有熔断，会持续打挂掉的 API**
  若 GitHub 整体故障，所有任务都在重试——
  **既浪费配额，又加剧对方的故障**。
  方案没有熔断机制。

- [ ] **重试日志会泄露敏感信息**
  §1 的 catch 若打印错误详情，可能含 **token**（GitHub API 的 401 响应）。
  项目 `log.ts` 有敏感键黑名单，但错误对象里的嵌套字段**不在键名匹配范围**。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
