# SRE 评审意见 — 通知中心「分类 + 批量操作」方案

> 评审对象：`tasks/n67-notification-center/environment/proposal.md`（31 行）
> 评审视角：**SRE（运维）** — 容量、可用性、故障域、变更安全、可观测性、回滚

## 摘要（前 5 条最重要）

1. **【严重】方案已是现存实现，等于重复立项、且是降级重写** — `src/app/api/me/notifications/read/route.ts:6,37` 已支持 `{all:true}`；`src/server/modules/notifications/index.ts:573-578` 已实现批量已读；`notification-list.tsx:64-74,113-120` 前端「全部标为已读」已上线。方案第 2、3 节的 SQL 与 `notifications/index.ts:575`、`457` **逐字相同** — 建议：作废本方案，改为在现有 `markRead(userId,{all:true})` / `GET ?unread_only=1` 上做过滤与分页增量，避免 SRE 为一次"已存在的功能"排发布与回归窗口。

2. **【严重】未读数"实时"承诺会打穿单进程同步 SQLite 的容量墙** — 方案第 30 行"未读数实时"；现状 `session-nav.tsx:62` 每次页面加载打一次带 JOIN 的 `/api/me/notifications?limit=1`，而 `notifications/index.ts:470-484` 的列表在**拉满 1 条记录**的同时还要 JOIN `users/content_entities/comments/comment_revisions` 四表。改成轮询/实时后 QPS = 在线人数 × 频率，全部压在 `db.ts:67-74` 的**单例同步连接**上（better-sqlite3 阻塞事件循环）— 建议：角标单独走一个无 JOIN 的纯 `COUNT(*)` 端点（`notifications/index.ts:456-459` 已有该语句可直接复用），并由前端按「标签页可见 + 降级间隔 + 后台暂停」调度，不做实时推送。

3. **【严重】批量已读是无上限的写放大，官方限流表里根本没有它** — `notifications/index.ts:573-578` 的 `{all:true}` 是**一次 UPDATE 打满该用户全部历史未读**，而 `rate-limit.ts:46-55` 的 `RateLimitAction` 九个动作中**没有 notifications 一项**，`api/me/notifications/read/route.ts` 全文无 `enforceRateLimit`（对照 `api/reports/route.ts:51` 等写接口都有）— 建议：新增 `notifications_read` 限流动作（如 30 次/分钟，`RATE_LIMIT_RULES` 需同步加 specKey），否则任何一个脚本都能把 WAL 写盘推满。

4. **【严重】批量操作没有按 kind 分类的能力，也没有保留策略，通知表只增不减** — `grep "DELETE FROM notifications|retention"` 全仓 **0 命中**；方案第 9 行"按 kind 分组展示"只解决展示，不解决增长。而索引 `idx_notifications_inbox(recipient_id, read_at, created_at DESC, id)`（`0041_role_notifications.sql:69`）**不含 kind** — 建议：本方案必须补 (a) 按 kind 过滤的服务端 WHERE（否则前端分组仍需整页拉全量再丢弃）、(b) 保留策略（如已读 90 天/每人 500 条上限，随每日 backup timer 同窗口跑），并在 `backup.mjs:84` 的备份清单里明确保留后仍可追溯。

5. **【严重】"按 kind 分组"踩在一张被重建过 5 次的表上，且新增 kind 需全表重建 + 关外键** — `notifications` 已被 `0013/0016/0034/0036/0037/0041`（6 次）用「FK OFF → BEGIN → 建新表 → 拷数据 → 删旧 → 改名」重建，每次都是 `-- dsh:migration-transaction=off`（`0041_role_notifications.sql:32`）；`migrate.ts:97-115` 明确写着**执行器不替它回滚，中途失败留半成品**。方案若因分类需要引入 new kind 或新列，就是第 7 次全表重建 — 建议：先在 `migrate.ts` 补"重建失败可恢复"的演练（备份 + `PRAGMA foreign_key_check` + 计数核对），再排发布；或改为不碰 CHECK 的加法（只加索引/只加查询参数）。

---

## 详细意见

### A. 容量与性能（P0）
| # | 问题 | 依据 | 建议 |
|---|---|---|---|
| A1 | 未读数与列表共用端点，拉 1 条也付 4 表 JOIN 的成本 | `notifications/index.ts:470-484`（`list()` 固定 JOIN；`limit` 只影响 LIMIT 不影响计划） | 拆出 `GET /api/me/notifications/unread-count` 只跑 `COUNT(*)`（语句见 :456-459），并给该路径加 nginx `proxy_cache` 短 TTL 或无状态缓存 |
| A2 | 分类筛选若在前端做，等于把每页 50 条再按 kind 丢，翻页会"越翻越少" | `notification-list.tsx:50` 固定 `limit=50`，无游标 | 服务端加 `kind=` 参数与 `(recipient_id, kind, created_at DESC, id)` 复合索引；分页改游标（现有 `api/_lib/pagination.ts` 可复用） |
| A3 | 单进程同步 SQLite 无水平扩展余量 | `db.ts:67-74` 进程级单例 + `server.mjs` 单实例；`compose.yaml` web 无副本概念 | 在方案里写明**容量上限口径**（如在线 500 人、每人 30 s 轮询 ≈ 17 QPS），超限的降级路径写清楚 |

### B. 变更与发布安全（P0）
| # | 问题 | 依据 | 建议 |
|---|---|---|---|
| B1 | 无滚动/灰度：批量操作上线即全量生效 | `deploy/dsh-review-web.service:20-21` `Restart=always` + 单实例，发布即重启 | 批量已读做成**后端开关**（`site_settings` 已有 `booleanSetting`，见 `_lib/context.ts:42`），先对 1% 用户开 |
| B2 | 迁移无事务保护区的失败处置没有 runbook | `migrate.ts:100-115`（schema 已变/登记失败 → 启动失败）；`compose.yaml:27-42` migrate 是一次性容器，`restart:'no'` | 若本次需要迁移，方案必须附**回滚 runbook**：先 `scripts/backup.mjs` 打点 → 记 `schema_migrations` 当前最大版本 → 失败时用 `scripts/restore-backup.mjs` 回退 |
| B3 | 备份与迁移窗口未协调 | `deploy/dsh-review-backup.timer:12` 每日 03:00（`RandomizedDelaySec=5m`），`backup.service:26` 用 `docker compose run` | 迁移发布避开 02:45–03:15；否则备份可能正好读到重建中途（虽有 WAL 快照，但仍是"半成品库"） |
| B4 | 双副本/双形态不一致 | 仓内**同时存在** `deploy/*.service`（systemd，`/srv/dsh-plugin-review`，node 直跑、PORT=80）与 `docker/compose.yaml`（容器，UID 10001、PORT 3000） | 方案里明确"按哪一套发布"；若两套都在用，批量操作的路由必须先在两套环境各走一遍 `scripts/acceptance-notifications.mjs`（`DSH_ACCEPTANCE_BASE` 可指任一端） |

### C. 可观测性与验证（P1）
| # | 问题 | 依据 | 建议 |
|---|---|---|---|
| C1 | 现有验收只覆盖"逐条已读 + 幂等"，**不覆盖 `{all:true}`** | `scripts/acceptance-notifications.mjs:12,134-138`（只有 ids 路径） | 方案必须补：`{all:true}` 的 200/幂等重放、越权（他人 all 只影响自己）、空集合、超 100 条 ids 的 400 |
| C2 | 无指标/日志埋点，批量操作事后不可归因 | `notifications/index.ts:571-591` `markRead` 无任何 `log()`；`rate-limit.ts:40` 已引入 log 但此处未用 | 批量已读记一行日志：`request_id + userId + marked + 耗时`，与 `proxy-common.conf:32` 的 `X-Request-Id` 贯通，便于从用户报障直接 grep |
| C3 | 健康检查不感知通知积压 | `health.ts:4-10` 明确"不做业务统计"（这是对的） | 保持 health 洁净，另在 `scripts/diagnose.mjs` 增一项通知表行数/最大未读用户，供运维巡检 |
| C4 | 无压测口径 | 全仓 `scripts/` 无通知压测脚本 | 上线前给基线：`{all:true}` 在 10 万行库上的 P95 耗时（同步驱动会直接体现为全站卡顿），写进方案 |

### D. 安全与越权（P1）
| # | 问题 | 依据 | 建议 |
|---|---|---|---|
| D1 | 批量已读缺限流（重复摘要 3） | `rate-limit.ts:46-55` 无 notifications 动作；`read/route.ts` 无限流调用 | 见摘要 3；另建议 nginx `limit_req` 兜一层（`site-locations.conf` 当前无任何 `limit_req_zone`） |
| D2 | 方案第 13 行**新造了一条与现有路由并存的写路径** | 方案 `POST /api/me/notifications/read-all` vs 现有 `POST /api/me/notifications/read {all:true}`；现有路由:`29-31` 已有互斥校验 | 禁止新增等价端点（双写路径 = 双份越权面 + 文档 `docs/API端点与路径.md:159` 漂移）；直接复用 |
| D3 | 方案 SQL 无参数绑定上下文 | 方案 :15-17、:24-26 用 `?` 占位是对的，但示例省略了 `assertCanWrite` 与错误映射 | 落到实现时必须走 `currentContext()` + `assertCanWrite`（`read/route.ts:23-24` 的口径），方案需写明 |

### E. 文档与运维配套（P2）
| # | 问题 | 依据 | 建议 |
|---|---|---|---|
| E1 | `docs/API端点与路径.md:158-159` 只写了两行，未记 `unread_only/limit` 参数与 `{all:true}` | 同上 | 方案落地时同步更新该文档，避免运维按文档排查时对不上参数 |
| E2 | 方案"预期"只有两句验收话术 | 方案 :29-31 | 补可执行验收：`scripts/acceptance-notifications.mjs` 增 3 条断言 + 一条 `EXPLAIN QUERY PLAN` 断言（确保分类查询走新索引，不出现 `SCAN notifications`） |
| E3 | 无「上线后如何判断好/坏」的指标 | 全方案 | 定义 3 个运维指标：通知表行数增速、`{all:true}` P95、`/api/me/notifications` 5xx 率（目前无采集） |

---

## 结论

**建议：打回重写（不是因为想法错，而是因为它没落在现有系统上）。**

方案描述的三件事——批量已读、未读数、按 type 分组——其中**前两件已在生产代码里完整存在**（`notifications/index.ts:571-591`、`session-nav.tsx:62`、`notification-list.tsx:64-74`），方案给出的 SQL 与现实现逐字一致。这意味着本次立项真正的增量只有"按 kind 分层展示"和"筛选"，而它带来的全部运维成本（迁移重建、限流缺口、无保留策略、无指标）在原方案里一条都没写。

重写时必须包含 **5 件事**，缺一不可：
1. 明确复用 `markRead({all:true})`，不新增 `read-all` 端点；
2. 角标与列表拆端点，角标用无 JOIN 的 `COUNT(*)`；
3. 补 `notifications_read` 限流动作并接到 `rate-limit.ts`；
4. 补通知保留策略 + `kind` 侧索引；
5. 附迁移回滚 runbook（备份打点 → schema_migrations 核对 → restore-backup）与避开 03:00 备份窗口的发布窗口。

> 补充说明：本次评审**只依据仓内代码与配置静态阅读**，未执行压测、未跑迁移、未连生产库——摘要 1/3/4/5 的结论来自代码路径与索引/DDL 的直接比对，A1/A3、C4 的耗时数字属**待验证假设**，落地前需用 `scripts/acceptance-notifications.mjs` + 一次真实库 `EXPLAIN QUERY PLAN` 证实。