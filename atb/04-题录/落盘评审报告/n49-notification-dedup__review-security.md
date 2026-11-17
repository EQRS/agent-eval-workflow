# N-49「通知去重与历史清理」方案评审 —— 安全工程师视角

> 评审对象：`tasks/n49-notification-dedup/environment/proposal.md`（31 行）
> 评审面：去重键的可信性、通知作为事件证据的可追溯性、删数据的不可逆面、执行载体的攻击/误伤面
> 结论：**不可按现状实施**。§1 与仓库现状三处正面冲突（会造成安全类通知静默丢失）；§2 是本部署面（单机 SQLite，无导出通道）上唯一的整表销毁入口；§3 的执行载体在仓库里不存在。

## 摘要（前 5 条最重要）

1. 【严重】§1 的 `dedupKey = recipientId:kind:contentId` 与既有 `UNIQUE (deduplication_key)` 形成两套键，`INSERT OR IGNORE` 会把「认领被驳回 / 身份申请被驳回」这类安全通知**静默吞掉** — 依据：`migrations/0041_role_notifications.sql:43,56`；`src/server/modules/notifications/index.ts:118,125,159,184` — 建议：不造第二套键，沿用 `deduplication_key` 单一权威；确需写入时显式 `ON CONFLICT (deduplication_key) DO NOTHING`，**禁止 `INSERT OR IGNORE`**（它会对整行任意约束失效，违反"约束即安全兜底"的项目纪律）。
2. 【严重】方案 INSERT 的列名与取值在仓库不成立，修复时会直接撞上两条安全 CHECK — 依据：`migrations/0041_role_notifications.sql:42`（真实列是 `content_entity_id`，方案写 `content_id`）、`:47-48`（`created_at INTEGER CHECK (created_at >= 0)`，方案用 `Date.now()`）、`:49`（`CHECK (recipient_id <> actor_id)`）、`:55`（栏目类通知必须有 `detail`/不能缺内容锚点）— 建议：方案里写清"必须经 `notifications` 模块的 `deliverXxxNotification` 投递"，不允许业务代码自拼 INSERT；仓库已有 `src/server/platform/time.ts` 作为时间唯一源。
3. 【严重】§2 无差别按 `created_at` 删 90 天 = 当前唯一的整表删除入口，而本仓**没有导出/通知补看通道**，删了就不可举证 — 依据：`src/server/modules/notifications/index.ts:442-449`（服务面只有 list/countUnread/markRead，无 delete）、`src/server/platform/db.ts:5`（外键纪律）、全仓 `grep 'dsh export|\.backup\(' src → 0 命中`、`docs/项目规划/架构设计.md:2139`（未读数由通知记录计算，回执/保留期限仍待确认）— 建议：删除条件收紧为 `read_at IS NOT NULL AND created_at < ?`；每次删除结果落 `operation_records`（该表由 0042 触发器保证不可改不可删，但**不记录 DELETE 之外的事实**——通知行本身没有第二份账本）。
4. 【高】§3「挂在 worker 每日任务里」在本仓无载体 — 依据：`src/worker/main.ts:178-188`（`runOnce` 里只有 `ensureDailyDiscoveryTask` 一条排程）、`src/worker/scheduled.ts:1-18`（明确"当前只有每日插件目录自动发现"）、`src/worker/main.ts:89-102`（未注册 handler 直接 `no_handler` 跳过，任务会被静默吃掉）、`packages/common/field-rules` 的 `TaskType` 是闭集 — 建议：复用 `DISCOVERY_SCHEDULE_CHECK_MS`（60s）那套"按天幂等 key + 每次启动试排"的现成骨架，但**不要复用 discovery 的 dedup key 命名空间**；或者不新建 task_type，直接在一个已注册 handler 的收尾钩子里跑（并写明为什么）。
5. 【高】§1 与 §2 的语义耦合错：去重行被删 → 事件重放 → 重发 — 依据：`src/server/platform/db.ts:48-50`（`foreign_keys = ON` + `defer_foreign_keys = ON`，所有 `ON DELETE RESTRICT` 生效）；`migrations/0037_forum.sql`（forum 回复是硬删语义，`src/server/modules/notifications/index.ts:524-547` 明确"帖子/回复被硬删"分支）— 建议：去重键的保留期必须 ≥ 业务事件存活期，或把"事件幂等键"与"展示行"分离（事件键表不可删，展示行按保留期删），否则会出现"回复被删→通知已清→内容重放→给一个已无关用户再发一条"。

## 详细发现

### 【严重】S-1 两套去重键并存：UNIQUE 只护一套，另一套失效即静默丢通知
- 现状：唯一权威键是 `deduplication_key TEXT NOT NULL CHECK(trim<>'') + UNIQUE`（`migrations/0041_role_notifications.sql:43,56`），且**每个 kind 自带不同键形态**：评论 `comment_published:<commentId>:<recipient>`（notifications/index.ts:118）、处理结果 `report_handled:…`（:149 注释）、精选 `featured:<期号>:<收件人>`（:207）、反馈 `feedback:<ticketId>:<recordId>`（:306）、身份 `role_application:<申请>:<收件人>:<结果>`（:353）、认领 `plugin_claim:<认领>:<收件人>:<结果>`（:375）、讨论区 `forum_reply:<replyId>:<recipient>`（:292）。
- 方案的 `recipientId:kind:contentId` 对上述**一个都对不上**，而且 collide 面正好落在"驳回→补充材料→再通过"这条安全语义上（`notifications/index.ts:353-354` 明确说这两种结果是两次不同结果，各应发一条）：两条消息 kind 同为 `role_application_result`、content 同为 null → 第二条被 IGNORE 吞掉。
- 安全后果：用户看不到"你的身份申请被驳回"，无法在期限前补材料；管理员也看不到"通知其实没发出去"（返回的是 `changes===0`，没人看）。
- 建议：① 删除本方案 §1 的键设计，统一走 `deduplicationKey`（**保留 kind 与 null 锚点信息**，如 `role_application:<id>:<recipient>:<outcome>`）；② 若坚持加复合键，唯一约束须改为 `UNIQUE(recipient_id, kind, content_id)` 并**同时重建 8 张迁移里的 key 形**——这属于表级破坏性迁移，必须走一次 `0008/0013/0016/0034/0036/0037/0041` 的并集口径检查（`tests/migrations-forum-union.test.ts` 就是为此存在的）。

### 【严重】S-2 INSERT 的字段/时间源/约束全是错的
| 方案 | 仓库事实 | 依据 |
|---|---|---|
| `content_id` 列 | 只有 `comment_id` / `content_entity_id` | migrations/0041:41-42 |
| `created_at: Date.now()` | 项目有唯一时间源 `nowMs()`（handlers 全用注入的 `now`） | notifications/index.ts:105-106,127-135 |
| 不写 `detail`/写 `comment_id` 空 | 栏目类 kind 必须 `detail IS NOT NULL`，且 `(kind IN ('moderation_result','featured_published',…)) = (detail IS NOT NULL)` 是**双向**等式 | migrations/0041:52-53 |
| actorId 可能等于 recipientId | `CHECK (recipient_id <> actor_id)` + 服务层先判 `self` | migrations/0041:49；index.ts:114,175,224 |

- 安全判断：这类"自拼 INSERT"绕过了通知模块里**所有**的语义守卫（self 判定、detail 非空、"栏目类必须有 link_url"见 index.ts:410-414）。通知是**用户可见的唯一安全反馈通道**，用一份会 ABORT 或会静默的 SQL 去逼近它，收益是"少发几条"，代价是"该发的没发"。
- 建议：方案改为「在 `notifications` 模块内加一个薄封装 / 或在 modules/notifications 上扩参数」，并在方案里写明"禁止绕过模块投递"的约束与对应的静态门禁（仓里已有 `scripts/check_local_paths.py` 一类门禁的思路，可同型加一条 grep 门禁：`src/**` 内不得出现 `INSERT INTO notifications`，除 `modules/notifications/index.ts`）。

### 【严重】S-3 90 天无差别删除 = 不可逆的举证缺口
- 仓库现状：`operation_records` 是唯一"不可改不可删"的账本（`migrations/0042_audit_immutable.sql` 触发器；`docs/现状-对规划-差距-2026-10-19.md` 也把审计列为已实现），而通知**不在**账本里。删了就只剩 `operation_records` 里"系统在某天改过数据库"的间接痕迹。
- 攻击面：一个被驳回的认领/申请，当事人主张"从未收到通知"。现状：可查 `notifications` 行。清理后：**无法举证**（剩下的是 dedup key 也已消失）。
- 数据最小化反向：未读通知对用户是"未消费"的信息，删它违反最小必要性里的"必要"一侧——保留未读、清已读才既减量又不丢义务。
- 建议：
  1. 删除条件 `WHERE read_at IS NOT NULL AND created_at < ?`，未读永不过期（或 365 天且必须先发"通知即将过期"的二次通知）；
  2. 每次清理写一条 `operation_records`（action_type=`notification_retention`，operation_key=当天日期），记录 removed 行数；
  3. 方案必须给一个**备份/导出**口径：目前 `grep 'dsh export|\.backup\(' src` 零命中（`api/admin/backups` 只有备份目录保留策略），也就是说"删了买不回来"。要么先在部署文档里写清备份含 db 且有可恢复演示，要么在方案里明确"删除即最终"。

### 【高】S-4 §3 执行载体不存在，且失败是**静默**的
- `src/worker/main.ts:178-188`：`runOnce` 每 60s 只做一件事——`ensureDailyDiscoveryTask`；失败只 `log.error` 后继续。
- `src/worker/main.ts:89-102`：未注册 handler → `errorCode: 'no_handler'`、`log.warn`、任务被跳过并计入 `attempt` —— 一个新 task_type 若没有 handler，会被当"失败任务"反复领取，直到 `retry_wait` 冷却，与用户任务强共享队列与并发上限（`maxConcurrent`，main.ts:190-195）。
- `scheduled.ts:11-12` 自己承认：worker 整天没跑就**跳过当天**，不补。清理任务用同一骨架可接受（幂等只是重复删），但方案必须写。
- 建议：明确"不新增 task_type"，把清理挂在现有 60s 排程钩子里、用天粒度幂等 key（`notification_retention:YYYY-MM-DD`）；并写清"worker 不在线导致延期清理"是可接受偏差。

### 【高】S-5 去重键的保留期与被引用业务实体的生存期不一致
- 讨论区回复是硬删语义（`src/server/modules/notifications/index.ts:524-547` 专门处理"帖子/回复被硬删"），删除后通知行仍留着（含 `detail` 摘要，即**被删内容的一份副本**）。90 天清理在这里反而是"补救"，但会与评论类通知撞：评论走后 `comments` 的可见性判定（`resolveVisibility`，`tests/comments.test.ts:6`），`content_entity_id` 仍在。
- 两侧都要回答：① 被删内容的摘要留在通知 `detail` 里算不算残留副本（安全上有"删除范围"的要求）；② 去重行被清后，若同一 key 被重放（`deduplicationKey` 由业务参数拼，`comment_published:<commentId>:<recipient>` 天然稳定，除非 comment 被重建，comment id 不复用 → 风险低；但 `featured:<期号>:<收件人>` 与 `role_application:…:<结果>` 属"人给的字符串"，**期号/申请 id 若被复用就会重发**）。
- 建议：清理任务里区分两类——"可重建锚点"与"不可重建锚点"；对讨论区/被删内容详情，清理粒度应是"抹掉 detail、保留行"还是"整行删"，方案必须给出选择理由。

### 【中】S-6 通知列表读取面与清理的一致性
- `notifications/index.ts:455-460`：未读数 `COUNT(*) WHERE recipient_id=? AND read_at IS NULL` 随行删除自然收敛，正确。
- `notifications/index.ts:557-559`：栏目类 `href` 取 `link_url`（精选/反馈/身份/认领都有 link_url），讨论区从 dedup key 第二位解析 `replyId` —— **如果清理删行，什么都不会坏**；但如果将来有人"为省空间把 dedup key 改短"，讨论区跳转就会静默失效。建议方案写明：`deduplication_key` 是 **业务锚点的承载列**（不是纯技术列），改它的形态必须带回归用例（`tests/forum-misc.test.ts` 有通知跳转回归）。

### 【中】S-7 令牌/凭据感染面（顺带核查，未发现新问题）
- 通知 `detail` 是自由文本，来源是审核人输入与自动摘要；当前没有把它写进日志或外部服务（`src/server/platform/log.ts:1-6` 规则禁止记私有正文），**本方案不引入泄漏面**。但清理任务日志里若打出被删行示例，会带出用户昵称/正文摘要，需注意只打 count。

## 建议的修订版 §1 / §2 / §3（可直接替换）
```text
1. 去重：不新增键。所有通知一律经 modules/notifications 的 deliverXxx 投递，
   唯一权威 UNIQUE(deduplication_key)，键形态按 kind 固定并在模块注释里登记。
   新增 kind 必须同步：① 迁移重建 CHECK 的并集口径（含 0037/0041 的 10 项）
   ② NotificationKind 联合类型 ③ tests/migrations-forum-union.test.ts 的并集用例。
2. 清理：DELETE FROM notifications WHERE read_at IS NOT NULL AND created_at < ?，
   默认 365 天（删前 30 天给未读用户发"将要归档"的二次通知），未读永不删。
   每次清理写一条 operation_records（removed/kept 计数），可追溯。
3. 载体：不新增 TaskType。挂 worker 的 60s 排程钩子，天粒度幂等键
   notification_retention:YYYY-MM-DD；worker 离线则顺延，不补跑历史。
```

## 与既有纪律的对照
- 约束即安全：项目把"自己不通知自己""栏目类必须有 detail/link_url"写成 DB CHECK + 服务层双判；方案的自拼 INSERT 同时绕过两层 → 应被"通知唯一通道"门禁挡住。
- 不可删账本 vs 可删通知：`operation_records` 有 0042 触发器，通知没有对应替代，删通知必须留审计。
- 时间唯一源：`created_at: Date.now()` 与 `nowMs()` 并存会在时钟跳变时破坏排序与清理边界（`idx_notifications_inbox` 是按 created_at DESC 排的）。
