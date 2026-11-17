# n46-ownership-dispute SRE（运维）评审

> t48 · m-sec · 2026-11-17

---

SRE（运维）评审完成，交付：atb/tasks/n46-ownership-dispute/output/n46-ownership-dispute-SRE评审.md（137 行，9 节）。

结论：不建议按现方案实施——方案把「争议处理」写成「一个表 + 一个 UPDATE」，而这件事的运维价值全在「过程可追溯」上。

8 条意见（均带 文件:行号）：
1) 「直接改归属人」在数据库层不成立（proposal:25-26）：撞 uq_plugin_ownerships_verified（0040_user_roles.sql:119-120，部分唯一索引）→ SQLITE_CONSTRAINT → errors-http.ts:120-126 只映射 DomainError → 500。这条 500 本仓已真实踩过并写进注释（ownership.ts:239-250）；正确做法在 ownership.ts:327-337（先查 taken 给人话再改）。仓库有既定转移路径 ownership.ts:313-367，方案绕开了它。改归属还会让原归属人开发者资格失效（service.ts:444 / ownership.ts:369-376），但 user_roles 不会自动撤销 → 悬空状态无人监控。
2) 无审计（proposal:25-26）：operation_records 是唯一审计载体且被 0042:17-27 锁成不可删改；identity/admin.ts:23-25 的惯例是每次变更同事务写一条。现有 reviewClaim（ownership.ts:313-367）也没写审计——方案把它变成日常操作后，站点将首次出现「关键权限变更完全无审计」。
3) status 无 CHECK（proposal:17）：本仓同类表全带枚举（0040:27/52-53/87-88）；脏状态行会永远留在 pending，且缺终态定义（应补 resolved_kept / resolved_transferred）。
4) 无配额、无前置校验（proposal:22-23）：工单量上界 = 插件数 × 用户数；RateLimitAction（rate-limit.ts:46-55）无争议类；还会被用来枚举「无主插件」。建议只允许对已有 verified 归属人的插件提争议 + 每天 3 次频控 + 对外统一「已受理」。
5) 无查重（proposal:12-19）：照 0040:69-70 加部分唯一索引 (plugin_id, claimant_id) WHERE status='pending'；另加队列索引（对齐 0040:115），否则管理员队列全表扫。
6) 迁移产物缺失（proposal:11-19）：未登记 REQUIRED_TABLES（migrate.ts:142-237）→ health.ts:76-82 门禁覆盖不到，表缺失落到首次提交才炸（0037 那批表的教训见 migrate.ts:214-215）。编号 0043 正确。
7) 连锁效应未定义（proposal:25-26）：撤销旧+建立新+写审计+发双向通知必须同事务（对齐 ownership.ts:173、tx.ts:42-52）；developer 身份是否随撤销需要产品决策。
8) 预期不可证伪（proposal:28-29）：「归属可纠正」其实不可纠正 → 给 6 条可测断言（不是 500、恰好 1 行 verified、恰好 1 条审计、查重命中、429、required_tables 含新表）。