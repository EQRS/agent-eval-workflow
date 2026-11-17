# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **前端禁用按钮不能防重复提交**
  §1 只是 UI 层。用户直接调 API、或者网络重试，都会绕过。
  前端防护**不是安全边界**。

- [ ] **内存 Set 在多进程下失效**
  §2 的 `inFlight` 是**进程内**变量。项目 web 与 worker 是两个进程。
  且进程重启后 Set 清空。

### 考点识别（③ 无用功）
- [ ] **项目已有 requestKey 幂等机制，41 个文件在用**
  `src/` 下 41 个文件使用 `requestKey`。
  且 `migrations/0001_baseline.sql` 有 `draft_save_receipts` 表
  （`PRIMARY KEY (draft_id, request_key)`、`CHECK (result_version = expected_version + 1)`）。
  **方案是在重造一个更弱的机制。**

- [ ] **operation_records 的 UNIQUE 也是幂等载体**
  `UNIQUE(action_type, operation_key)`，且 `0042` 给了防篡改触发器。

### L4 系统性
- [ ] **内存 Set 会在异常路径泄漏**
  §2 的 `try/finally` 看似正确，但若 `inFlight.add` 之后、
  `try` 之前进程被 SIGKILL，Set 状态随进程消失——**这不是 bug，是它证明自己没意义**。
  真正的问题是：多进程 + 重启 让进程内状态**天然不可靠**。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 考点识别（关键，2 分）

- [ ] **是否识别出这是「无用功」（方案要解决的问题已解决）或「倒退」（已有更好实现）？**
  若识别出，需明确指出仓库中已有的实现位置。

## 超纲发现

- [ ] 预设之外、经核实成立的问题
