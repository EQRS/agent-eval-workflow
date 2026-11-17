# n57-transaction-wrapper SRE（运维）评审

> t42 · m-perf · 2026-11-17

---

SRE（运维）评审（n57-transaction-wrapper proposal.md，41 行）：**方案不能按现状实施 —— 在这个技术栈上物理上做不到**。6 条（2×P0 / 2×P1 / 2×P2），全部实跑验证。

**总评**：better-sqlite3 是同步驱动，一个进程只有一条连接、BEGIN 是连接级全局状态（db.ts:66-74 的 getDb() 是进程级单例）。把事务体改成 async，等于在 await 处把写锁让给**同进程的其他请求**。这不是可运维性问题，是**会静默丢数据**的问题。

**P0-1 await 期间事务漏给其他请求**：实测请求 A 开事务→插 A1→await 5ms，期间请求 B 在同一连接插 B1，A 再插 A2 并 COMMIT，最终表内顺序 ["A1","B1","A2"] —— **B1 被 A 的事务吞了**。反向更狠：若 A 在 await 之后抛错触发 ROLLBACK，**B 已回给用户「保存成功」的写入会被一起回滚**，B 毫不知情。better-sqlite3 官方要求事务回调同步；现有 tx.ts:37-52 的注释正是「回调必须是同步函数」。→ 保持同步事务，用 tx.run/tx.get 这类同步小助手改善可读性；需要 await 的按既有纪律在事务外做完。

**P0-2 嵌套直接报错，而嵌套是常态**：全仓 withTransaction 调用点 **161 处**，存在 workflow→module 的跨模块嵌套；db.transaction() 有 savepoint 嵌套语义所以现在能跑，手写 BEGIN 实测报 "cannot start a transaction within a transaction"；即使不报错，内层 COMMIT 会提前提交外层，原子性被打破。→ 保留 db.transaction() + makeTx() 的 finished 守卫。

**P1-3「支持异步操作」是伪需求且引入长事务**：最容易写出的就是在事务里等网络；SQLite 长事务是全局写锁，web 与 worker 两进程（compose.yaml:44-76、:78-108）会互相堵在 busy_timeout=5000（db.ts:56）超时后 500。现在写事务实测 200×50 行 synchronous=FULL 仅 165ms（≈0.8ms/事务），放进 await 后事务时长从亚毫秒变成网络 RTT。→ 建议把「事务内禁止 await」做成可执行约束（检测 AsyncFunction 直接抛），长流程用状态机+短事务。

**P1-4 无超时兜底**：try/catch 只处理抛错、不处理永不到达。卡住的事务永久占写锁 → 所有写请求 busy 超时 500；web 的 HEALTHCHECK 要 interval 30s × retries 3（web.Dockerfile:76-77）才判 unhealthy，即最长 90 秒完全不可用窗口。且全仓**没有任何 BUSY 统计或告警**。→ 保持同步；若坚持 async 必须 Promise.race 超时 + ROLLBACK + log.error 带 request_id。

**P2-5 161 处机械替换无灰度**：牵动事务边界、错误传播、返回值形态，且很多调用点直接 return withTransaction(...)，改 await 会强迫整链变 async；事务不可部分回退，出问题只能回滚发版。→ 不做全量替换，按模块灰度 + foreign_key_check。

**P2-6 ROLLBACK 自身失败会掩盖原始错误**：实测 "cannot rollback - no transaction is active" 会替换掉真正的业务错误；现有实现用 try/finally + markFinished 并有「事务已结束」守卫（tx.ts:24-27），方案没有对应保护。

完整评审：`_work/reviews/n57-transaction-wrapper.sre-review.md`（130 行）。临时探针目录已清理，未改动仓库源码。