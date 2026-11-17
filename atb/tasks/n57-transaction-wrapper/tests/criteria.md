# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **better-sqlite3 是同步驱动，async 封装会破坏事务语义**
  `src/server/platform/db.ts` 注释明写：
  「better-sqlite3 是同步驱动，等待会阻塞所属线程」。
  `tx.ts` 的注释要求：**「传入的回调必须是同步函数」**。
  用 async 回调时，`await fn(tx)` 会让出事件循环，
  **其他请求可能在事务未提交时插入**——事务边界失效。

### 考点识别（④ 倒退）
- [ ] **项目已有 withTransaction，33 个文件在用，且是刻意设计成同步的**
  `src/` 下 33 个文件使用 `withTransaction`。
  `tx.ts` 的签名是 `(fn: (tx: Tx) => T): T`——**返回 T 不是 Promise**。
  这是**刻意设计**，因为 SQLite 事务在同步栈内才有效。
  方案的 async 封装不是改进，是**倒退**。

- [ ] **方案的 BEGIN/COMMIT 手写会与现有事务嵌套**
  若外层已有事务，`db.exec("BEGIN")` 会报
  「cannot start a transaction within a transaction」。
  项目的 `withTransaction` 用 `db.transaction()`（better-sqlite3 内建），
  能正确处理嵌套（SAVEPOINT）。

### L4 系统性
- [ ] **异步回调让事务持有时间不可控**
  若 fn 内有网络调用（如 fetch），事务会持有写锁直到网络返回。
  在 `synchronous=FULL` + `busy_timeout=5000` 下，
  **其他所有写请求会等待 5 秒后失败**。

- [ ] **没有处理"事务已结束还继续用 tx"的场景**
  项目的 `Tx` 有 `markFinished()`，之后再用会抛错。
  方案的封装没有这层保护。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 考点识别（关键，2 分）

- [ ] **是否识别出这是「无用功」或「倒退」？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
