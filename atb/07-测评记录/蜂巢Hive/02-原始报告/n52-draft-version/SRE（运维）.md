# n52-draft-version · SRE（运维）

> 蜂巢 Hive · job h_蜂巢_终极重测_验证单元_0016
> 模型 deepseek-chat · 5907 字 · 20 次工具调用

---

已读完提案、相关迁移与关键代码。下面按 SRE 视角给出评审。

---

# SRE 评审：草稿保存的版本控制方案

**结论：方案在"正确性"这条线上方向对，但它把一段已知的、已在生产代码里解决了八成的 CAS 逻辑，重写成了一个更弱、更危险的版本 —— 并且删掉了所有可运维性设计。作为 SRE，我反对按当前文本上线。**

## 先说一个硬事实

提案里的 SQL 与仓库中**已经存在**的实现不一致。`src/server/modules/reviews/index.ts:631-651` 的 `saveDraft` 已经是：

```ts
UPDATE content_drafts
   SET payload = ?, save_version = save_version + 1, saved_at = ?
 WHERE id = ? AND save_version = ?
```

即 **CAS（比较后写入）**；影响行数 0 就是版本冲突。而提案 `proposal.md:11` 写的是先 `if (expectedVersion !== current.save_version) throw`、再无条件 `UPDATE ... WHERE id = ?`。

**这是一次从"原子"到"读-改-写（TOCTOU）"的倒退。** 提案等于要求把一个安全实现改回不安全实现。

---

## 1. 先查后写：并发窗口内会真的互相覆盖

**① 哪部分有问题**
`proposal.md:8-16`：`if (expectedVersion !== current.save_version) throw` 与 `db.prepare("UPDATE ... WHERE id = ?").run(...)` 是两次独立交互，中间没有原子性保证。

**② 后果**
- 两个设备同时自动保存，都读到 `save_version = 5`，都通过 `!==` 检查，都执行 `save_version = save_version + 1` → 最终 `save_version = 7`，**但只有一份 payload 存活，另一份被静默覆盖且版本号"看起来正常"**。这正是提案 `proposal.md:3` 声称要解决的问题。
- 更糟：版本号跳了 2，但没有任何一条冲突记录。事后追查"我的内容去哪了"时，**日志里什么都查不到**。
- 对照现实：`index.ts:643-651` 用 `updated.changes === 0` 判定冲突，是单条语句内的原子比较；`index.ts:9` 的模块注释明确把这条列为不变量。
- 且有 `CHECK (save_version >= 0)`（`migrations/0001_baseline.sql:120` 附近），但 CHECK 拦不住这种覆盖 —— 它只约束非负。

**③ 建议**
用条件更新，删掉先查逻辑：
```sql
UPDATE content_drafts
   SET payload = ?, save_version = save_version + 1, saved_at = ?
 WHERE id = ? AND save_version = ? AND deleted_at IS NULL;
-- changes === 0 → conflict
```
并附一条并发测试：两个调用同时以同 `expectedVersion` 保存，断言**恰好一个成功、另一个拿到 conflict**。

---

## 2. 提案的 UPDATE 漏了 `deleted_at IS NULL`：会把已提交/已删除的草稿"复活"

**① 哪部分有问题**
`proposal.md:13-15` 的 UPDATE 只按 `id` 定位。

**② 后果**
- `migrations/0018_draft_soft_delete.sql:14-19` 明确：已提交过的草稿是**软删除**（`deleted_at` 打标记），且"删除后不可再读写"。真实实现 `index.ts:191-196` 的 `readDraftRow` 带有 `AND deleted_at IS NULL`。
- 提案这条 UPDATE 绕过该过滤 → **一、仍然带着 `deleted_at` 标记的行被改写 payload**，破坏 0018 的"不可再读写"语义，也让提交/修订历史与草稿载荷脱节。
- 二、`0018:17-19` 的部分唯一索引 `uq_drafts_review` 只约束 `deleted_at IS NULL` 的行。往软删除行写数据不报错，但**业务上那份"已撤回的评测"的草稿内容被悄悄改成了新内容**，撤回语义（`withdrawn_at`）与草稿内容不再一致，审计时无法解释。
- 三、`save_version` 在已删除行上被递增，若后续有"按版本号做幂等/回溯"的逻辑，会出现语义空洞。

**③ 建议**
UPDATE 与冲突判定都加 `AND deleted_at IS NULL`；`changes === 0` 时要区分是"版本不符"还是"草稿已不存在/已删除"（前者 409 conflict，后者 410/不可用），否则删完草稿再点自动保存会收到误导性的"请刷新"提示。

---

## 3. 30 秒自动保存：没有退避、没有节流、没有开关，直接冲着频控和 SQLite 写锁去

**① 哪部分有问题**
`proposal.md:19-20`：「编辑器每 30 秒自动保存一次」。

**② 后果**
- **无条件的定时写**：用户开着 20 个标签页 = 40 写/分钟，全部打到同一个 SQLite 文件。SQLite 单写者模型下（WAL 也一样），这会与 worker、心跳（`migrations/0004_runtime_heartbeats.sql`）、审计落库（`migrations/0042_audit_immutable.sql` 说明 `operation_records` 是"幂等去重的唯一载体"，关键路径都依赖它）争抢写锁，表现为 `SQLITE_BUSY` 与整体延迟抬升。
- **无人值守时仍在写**：编辑器留在后台标签、用户出门——只要页面没关就一直 30s 一次。这既浪费写放大，也让"最后保存时间"失去意义。
- **与频控相互打脸**：`migrations/0006_rate_limiter.sql` 存在按动作键计点的限流表。自动保存若纳入限流，用户会遇到"自动保存被限流"这种荒谬体验；若不纳入，等于给限流开一个 30s/用户的旁路。
- **幂等缺失**：没看到 30s 自动保存如何处理"网络超时后重试"。真实代码在 `index.ts:585-600` 附近明确写了 `draft_save_receipts` 回执机制 —— **重放必须在 CAS 之前判定，否则带旧 expectedVersion 的重放会被误判成假冲突**。提案完全没提回执，离线补发/客户端重试会制造大量虚假冲突提示。

**③ 建议**
- 只在**内容有变化**（payload 摘要变化）时才发；无变化跳过（客户端比对 digest）。
- `document.visibilityState !== 'visible'`、或用户闲置 N 分钟时**暂停**自动保存。
- 冲突/失败后指数退避（30s → 60s → 120s，上限 5 分钟），并**暂停**而非持续重试。
- 自动保存走**独立的频控桶**，与手动保存分开计数，避免互相挤占。
- 复用仓库已有的 `requestKey` + `draft_save_receipts` 回执做幂等，不要另造一套。

---

## 4. 「冲突时提示刷新，丢失本地修改」—— 这是数据丢失，不是冲突处理

**① 哪部分有问题**
`proposal.md:22-24`：「提示用户刷新，丢失本地修改」。以及 `proposal.md:27` 的预期「不会互相覆盖」。

**② 后果**
- 方案自己承认会丢数据，却把预期写成"不会互相覆盖"。**这两句自相矛盾，且预期句是错的** —— 覆盖确实发生了，只是被改名为"冲突提示"。
- 从 SRE 角度这是**最贵的故障类型**：非崩溃型、无告警、无日志、用户事后才发现，而且我们**没有任何指标能发现它**（见第 5 条）。这类问题通常只能靠用户投诉发现，MTTD 以天计。
- 具体丢失量：一次 30s 周期内的输入全部作废。如果用户正在写长文，一次冲突可能丢掉整段。
- 真实代码的做法是 `index.ts:648-651` 「版本不符：不写回执、不改 payload，用户输入由调用方保留」—— 保留在**客户端**，配合 `current_version` 回传让客户端能基于最新版重试/合并。提案把这条最重要的可恢复性设计去掉了。

**③ 建议**
- 冲突时**不得清空本地编辑区**。至少：(a) 本地保留一份（localStorage/内存），(b) 展示"服务器版本 / 我的版本"二选一或做三方合并，(c) 提供"另存为副本"逃生口。
- 冲突响应带上 `current_version` 与服务器侧 `saved_at`（现有实现已带 `current_version`，见 `index.ts:649-651`），让前端能做自动 rebase。
- 把预期改成诚实的表述：**"冲突会被检出并保留双方内容，不静默覆盖"**。别写"不会互相覆盖"。

---

## 5. 完全没有可观测性与容量规划 —— 这是 SRE 视角最严重的一条

**① 哪部分有问题**
整份 `proposal.md`（28 行）**没有任何**监控、日志、指标、告警、灰度、回滚内容。

**② 后果**
上线后无法回答这些运维基本问题：
- 冲突率是多少？正常应该接近 0，若 >1% 说明自动保存太激进或客户端版本没更新。**没有指标 = 无法判断方案是否奏效**，`proposal.md:26-27` 的两条"预期"根本无法验收。
- 自动保存带来了多少额外写 QPS / 数据库增长？`content_drafts.payload` 是整篇 JSON 全量重写（`index.ts:645` `SET payload = ?`），30s 一次意味着**每篇草稿每天重写 ~2880 次**，每次写整篇正文。WAL 与备份体积的增长没有预估。
- `content_drafts` 只有 `idx_drafts_owner_active`（`0018:19`）这个 owner 维度索引；高频 UPDATE 会持续制造新页面版本，**对 `backups/` 目录（仓库里确有该目录）与 WAL checkpoint 的影响未评估**。
- 出错时没有任何排查抓手：冲突、写入失败、自动保存跳过，三者**在日志里无法区分**。

**③ 建议**
- 埋 4 个指标：`draft_save_total{outcome=ok|conflict|rejected|error}`、冲突率、自动保存写 QPS、`content_drafts.payload` 平均字节数。
- 冲突率设阈值告警（例如 5 分钟内 >2% 持续 10 分钟）。注意：这 30s 自动保存是否真的"高频"要等指标说话，所以**先灰度**。
- 结构化日志：每次冲突记 `draft_id`、`expected_version`、`current_version`、`actor`（user id），**但不记 payload 内容**。
- 明确回滚方式：前端开关一行关掉自动保存（服务端下发或构建期 flag），不需要发版。
- 灰度：先 5% 用户，观察冲突率与写放大，再放全量。
- 容量：评估「payload 全量重写」是否可以改为"仅在内容变化时写"（客户端算 digest 先行跳过），这是最低成本的手段。

---

## 6. 其他 SRE 关注点（次要，但需要在方案里定）

- **载荷无大小上限**：`proposal.md` 未提 payload 体积/分段数上限。`0018`/`0001` 的 `payload` 只有 `json_valid` 约束，没有 `length()` 上限。自动保存高频 + 无限载荷 = 单行膨胀与全库备份变慢。建议加 `CHECK (length(payload) <= N)`，并在服务层提前拒绝。
- **重放与 `save_version` 语义**：提案没有"同键同摘要返回原回执且不递增版本"的设计（真实实现见 `index.ts:585-600` 注释）。缺了它，客户端每次网络抖动重试都会**把版本号推高一格**，让其他设备的 conflict 概率凭空上升 —— 这是纯自伤的写放大。
- **`saved_at` 未更新**：提案的 UPDATE 只改 `payload` 与 `save_version`，没更新 `saved_at`（真实实现 `index.ts:645` 有 `saved_at = ?`）。运维靠 `saved_at` 判断草稿活跃度、做冷数据清理，漏更会让这个字段失真。
- **未提草稿清理策略**：30s 自动保存会显著增加"僵尸草稿"数量。没有 TTL/归档策略，`content_drafts` 会持续增长。建议明确清理窗口与保留期。

---

## 给方案作者的一句话

仓库里 `src/server/modules/reviews/index.ts` 的 `saveDraft` 已经把这件事做对了八成（CAS、回执幂等、`deleted_at` 过滤、保留用户输入）。**请以那份实现为准来改写提案，而不是另写一个更弱的版本**；然后补上这份提案完全缺失的部分：可观测性、退避/节流策略、灰度与回滚、以及诚实的冲突语义。补齐这三块我可以签字；按现在这版不行。