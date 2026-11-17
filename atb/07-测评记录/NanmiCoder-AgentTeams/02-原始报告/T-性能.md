# N-46 评审报告 · NanmiCoder T-性能

> 测评：NanmiCoder Agent Teams `@nanmicoder/dsh-agent-teams@0.1.30`
> 成员：e3-fixer · 任务：t5 · 状态：completed
> 日期：2026-11-17

---

N-46 性能视角评审完成。落盘：atb/tasks/n46-ownership-dispute/review-perf.md（231 行，只读评审，未改源码）。

结论：不能按现方案上线。方案只有 8 行 SQL + 3 句流程，通篇无一条性能约束；落在单文件 SQLite（better-sqlite3 单写者、读同步阻塞 Node 事件循环）上，代价被放大。

摘要（前 5 条）：
1.【严重】§2「任何用户 × 任意插件」无频控无前置校验 = 写放大炸弹。proposal.md:23；对照本仓 rate-limit.ts:46-55/68-134（9 项 RATE_LIMIT_RULES 无一覆盖），以及同类人工审核接口的标准写法 role-applications/route.ts:41-56（enforceRateLimit + refundQuietly 退还）。后果：每插件一行、每行一次写事务，占满 SQLite 全库写锁。
2.【严重】新表零索引（proposal.md:12-19 只有 PK），而唯一用途就是「按 status 筛 + 按 created_at 排」的后台队列。对照 0040_user_roles.sql:64-65/113-115 的既定写法（role_applications 与 plugin_ownerships 都建了 (status, created_at, id) 队列索引）。后果：每次打开待审页 = 全表扫 + 临时排序，同步阻塞事件循环。
3.【高】无 UNIQUE(plugin_id, claimant_id)，表无界增长。对照 0040:69-70 的部分唯一索引（pending 唯一）、0040:111 UNIQUE(plugin_id,user_id)。建议照抄「待审唯一、终态不唯一」口径。
4.【高】§3「直接改归属」未描述事务边界，必然踩 uq_plugin_ownerships_verified（0040:119-120），且现有 ownership.ts:239-250 注释记录了同类「撞唯一索引→500」的真实事故。一次变更 = 降级旧行+提升新行+审计 INSERT+通知 INSERT，4 次写在同一写锁内，方案没要求合并或定顺序。
5.【中】漏掉通知链路成本：notifications.kind 是硬 CHECK（0041_role_notifications.sql:39），加新 kind 要整表重建（0041:36-67，第 6 次重建，PRAGMA foreign_keys=OFF 窗口），而 compose.yaml:81-83 让 worker 等 migrate 完成 —— 迁移越慢站点就绪越晚。建议复用现有 plugin_claim_result kind。

完整清单 12 条（P-01…P-12），关键几条：
- P-01 频控：给 RATE_LIMIT_RULES 加 ownership_dispute（照 report / role_application 口径）+ settings 配置项 + refundQuietly。
- P-02 索引：给出可直接抄的 DDL（队列索引、plugin_id 索引、pending 部分唯一索引）+ evidence 长度上限（对照 0040:57/89 的 500 字约束）。
- P-03 唯一约束 + 队列必须 LIMIT 游标分页（不要 OFFSET；现有 listQueue ownership.ts:296-310 本身也没分页，照抄会踩同一坑）。
- P-05 审计缺失：operation_records 是唯一审计载体（0001:411-431，0042:17-27 已禁 UPDATE/DELETE 只许追加），漏写会让事后核查从「一条索引查询」变成「全库 diff」。
- P-07 status 无取值集合（对照 0040:87-88 / 52-53 的 CHECK 枚举）。
- P-08 evidence 无长度上限，若实现成塞 BLOB 会让表扫描成本与备份体积一起爆。
- P-09 未加进 migrate.ts:142-237 的 REQUIRED_TABLES，缺表时 /api/health 仍 200（migrate.ts:214-215 记录过同类事故）。
- P-10 无保留期，全表扫描随时间单调变慢；P-11 与既有认领队列查询重叠；P-12 归属查询应走 uq_plugin_ownerships_verified 而非全表扫。

两条落地建议：方案 A（保留新表）把 P-01~P-03/P-08/P-09 补齐即可接受，报告里给了完整 DDL；方案 B（不建表）把 plugin_ownerships.status 扩成 disputed，复用已有队列索引与 reviewClaim 事务骨架 —— 性能上严格优于 A，代价是一次表重建窗口。

前提核实：proposal.md:4 的「现状」与仓库一致（0040:76-81、ownership.ts:9-18），这一点方案没说错。