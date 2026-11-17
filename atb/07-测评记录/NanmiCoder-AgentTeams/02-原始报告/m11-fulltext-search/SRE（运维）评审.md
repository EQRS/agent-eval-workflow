# m11-fulltext-search SRE（运维）评审

> t12 · m-sec · 2026-11-17

---

SRE（运维）评审完成，交付：atb/tasks/m11-fulltext-search/output/m11-fulltext-search-SRE评审.md（210 行，14 节）。

结论：不建议按现方案实施。

SRE 专业面 13 条（均带 文件:行号 证据）：
1. 第12-17行未指定 tokenize → unicode61 中文子串搜不到（migrations/0014_review_search.sql:5-9 已实测存证），与第47行「支持中文」互斥，且静默返空。
2. 第19-31行无重建/回填入口；独立 FTS 表不能用 rebuild 命令，脱节后无修复手段；对照 scripts/backfill-review-search.mjs 是必跑步骤。全量重建撞 db.ts:53-56 的 WAL+5s busy_timeout，与 web/worker 两进程抢写锁。
3. 第23-30行 DELETE/INSERT 无事务、不与业务同事务 → 「更新」变「消失」，「搜得到但打不开」（对照 publish-review.ts:142-145 的同事务纪律）。
4. 第33-41行无 LIMIT/无超时/无分页 + better-sqlite3 同步驱动（db.ts:6-8）→ 一条热门词卡死整个 web 进程；HEALTHCHECK(web.Dockerfile:76-77) 会把它判死重启。且覆盖了既有的 /api/plugins/search（route.ts:20-21 原有的 q 截断与 limit 上限被去掉）。
5. 全篇无迁移产物：migrations/ 无文件 → 表不存在；未登记 REQUIRED_TABLES（migrate.ts:142-237）→ health db.required_tables 失败 → 503 → 编排不放流量（health.ts:76-82）。
6. 备份面：scripts/backup.mjs:78-88 行数清单是硬编码白名单，新索引表成盲区；docs/docker约定.md:163 推荐的宿主 sqlite3 在镜像里根本不存在（web.Dockerfile:14,19-21）。
7. health.ts:6-7 端点公网可达，新增 check 不能回显业务量；FTS 空索引时现有检查全绿（最恶劣静默故障）。给出 fts.* 三条指标，degraded 不判 503。
8. 「<50ms」无规模前提；trigram 索引膨胀未估算，与 health.ts:94 的 200MB disk.free 阈值 + 14 份备份副本抢同一块盘。
9. 第44行 bm25 排序在「一行一插件」下退化，且无列权重/无 tie-breaker → 顺序不稳定、不可复现。
10. 匿名高频只读接口无任何限流；rate-limit.ts:46-55/209-213 全是登录用户键，用不了；nginx site-locations.conf:39-44 无 limit_req。
11. 路径冲突：/api/plugins/search 已存在（catalog listPlugins + buildDirectoryWhere），LIKE 子串对中文天然有效——问题已被解决过。
12. 无变更/回滚/对账剧本（migrate 一次性进程、每日 03:00 备份、重建前必须先有恢复点）。
13. 最小补齐清单 9 项（按实施顺序，含验收方式）。

已排除误报：migrations 排序 localeCompare(b,'en') 下 0043 正常；nginx limit_req 未配置属未写而非写错（作为建议而非缺陷）。