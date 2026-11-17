# n30-n-plus-one SRE（运维）评审

> t33 · m-sec · 2026-11-17

---

SRE（运维）评审完成，交付：atb/tasks/n30-n-plus-one/output/n30-n-plus-one-SRE评审.md（154 行，10 节）。

结论：不建议实施——方案优化的是一个不存在的性能问题。列表页没有 N+1，分类早就是一次批量查询完成的。

8 条专业意见（均带 文件:行号）：
1) 前提不成立（proposal:3-19）：本仓 listPlugins（catalog/service.ts:784-897）是 COUNT + 1 条含 4 个相关子查询的聚合 SQL（:816-832）+ 1 条分类批量（:858-861）+ 1 条 overrides（:863），无任何 per-row 循环；「21 次 vs 2 次」的对比本身是错的（真实是 4 条）。
2) 假设的表结构不符（proposal:16,27-29）：plugin_categories 无 repository_id 列（0010_plugin_categories.sql:19-27），分类靠 topics 交集关联（service.ts:248-269）；照抄会 SQLITE_ERROR → 列表页全线 500。这是低审查强度改动造成高影响故障。
3) 真正瓶颈未被识别（proposal:33-35）：PUBLIC_REVIEW_COUNT_SQL / ASSESSMENT_TIER / SECURITY_AUDIT / ASSESSED_COUNT 四个相关子查询在 SELECT 列里**逐行执行**（service.ts:137-194, 822-825），且缺覆盖索引（0001_baseline.sql:159/179/147 各走不同路径）。优化分类查询只动常数项。
4) 零测量（proposal:5,33-35）：server.mjs:62-82 已有每请求 duration_ms + slow 标记（:66-67），方案不用；无基线 = 不可验证 = 不可回滚。且 better-sqlite3 同步驱动（db.ts:6-8），查询耗时直接相加，条数≠时间。
5) 空数组保护：方案缺，但本仓已正确处理（service.ts:250 空数组直接返回不发查询）——重写会把已处理好的边界重新变成 500，属功能退步。
6) SELECT *（proposal:13,25-29）：列漂移静默扩散；且 categoriesByRepository 只需 slug/label/sort_order（service.ts:254），SELECT * 会额外搬 topics JSON，I/O 放量。
7) 页面整体成本（page.tsx:175-194）：一次「筛选为空」的渲染串 7 条查询；:194 的第二次 listPlugins 只为拿 total 却跑全套聚合子查询——真正低成本高收益的优化点。
8) 缺变更/回滚剧本：页面 500 无法靠重启自愈（compose.yaml:68 restart unless-stopped），只有 revert 能救；需 p95/500 数与回滚判据。
另附建议的方案骨架（测量→归因→改造→对比→回滚）。