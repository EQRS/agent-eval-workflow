# n9-concurrency SRE（运维）评审

> t27 · m-perf · 2026-11-17

---

SRE（运维）评审（n9-concurrency proposal.md，34 行）：**方案不能按现状实施**。7 条（3×P0 / 2×P1 / 2×P2），全部用本仓库 better-sqlite3 + SQLite 3.53.4 双进程实跑验证。

**总评**：这是「把并发控制从数据库挪进应用进程」的反模式 —— CAS 只保护单行，而排序保护的是多行相对关系；重试无上限/无退避/无事务/无 409 映射，会把前端并发变成服务端忙等 + 静默数据损坏。

**P0-1 §3 无界忙等 = 可远程触发的 DoS**：实测 changes 恒为 0 时 while(true) 1 秒空转 **15,189 次**，单核占满；better-sqlite3 同步不可中断（db.ts:1-11 原文「事务保持短小」），事件循环被钉死后健康检查（web.Dockerfile:76-77，5s×3）判 unhealthy → 502 扩成整轮编排重启。触发条件（plugin_id 传错、行已删、ctx.userId 为 null）都不需要并发。→ 有界重试 3 次 + 退避 + err.conflict（errors.ts:87-88）；先判 row 是否存在（实测 undefined 静默，套 catch 即成死循环）。

**P0-2「顺序不会错乱」不成立**：表上无 UNIQUE(user_id,position)；实测双进程各 300 轮×5 行，**5 行全部 position=299**，零报错；且两行 DEFAULT 0 天然并列（实测 a(0),b(0),c(2)）；现有读取仍按 created_at DESC（interactions/index.ts:900、:960），position 并列时翻页重漏（catalog/service.ts:790-791「排序键必须唯一」）；迁移未回填，上线即全员并列。→ 唯一索引 + 稀疏步长 + 同事务整批落定 + 迁移内回填。

**P0-3 迁移不可回滚**：建列/回填/唯一索引必须同一份迁移（migrate.ts:64-128 每文件一事务；SQLite 不能删列），否则留下「列已加、全是 0」的不可回滚中间态；加唯一索引会持写锁，需给停机预估。→ 单迁移三件事 + RAISE(ABORT) 自检 + 发布前备份。

**P1-4 重试被过度设计**：实测双进程 1503 次尝试仅 **3 次** CAS 冲突、**0 次** SQLITE_BUSY（WAL + busy_timeout=5000 已串行化）；真正冲突来自前端每次拖拽一个请求；示例还**没包事务**。→ withTransaction 整批提交 + 前端防抖合并成一条整段顺序请求 + 409 重拉。

**P1-5「用户感知不到冲突」= 吞错误**：无 409 时落 internal→500+unhandled_error，把并发冲突误报成服务器故障；仓库已有 version_conflict（errors.ts:17-18）却未用；冲突率无法观测。

**P2-6 缺运维面**：空转循环没有 finish 事件 → 不触发 slow 日志（server.mjs:31/:66-80），故障完全隐形；排序接口无频控（既有写法 api/plugins/route.ts:67-73）；无重建/回滚预案。

**P2-7 写入口未交代**：addFavorite（interactions:648-650）与批量移动（:1114-1119）都不写 position → 新条目全拿 DEFAULT 0，立刻造成并列。

完整评审（含实测数据表 + 可复用约定对照）：`_work/reviews/n9-concurrency.sre-review.md`（150 行）。临时探针目录已清理，未改动仓库源码。