# 安全工程师评审：N-52《草稿保存的版本控制》

**评审对象**：`atb/tasks/n52-draft-version/environment/proposal.md`（全文 28 行）
**评审视角**：仅安全。架构正确性、产品体验另由他人覆盖。
**结论**：方案的**技术方向已被仓库现有实现完整覆盖且更严谨**，但方案本身作为"技术方案"存在 **1 条 P0（照抄即造越权）**、**3 条 P1**、**3 条 P2** 的安全缺口。建议**不要按本方案落地**，改为"补测试 + 补上限"，而非重写保存路径。

---

## 一、先说结论：方案第 1 节的 UPDATE 缺 WHERE 的 owner 条件（P0）

**① 哪部分有问题**
`proposal.md:15-17`：
```ts
db.prepare(
  "UPDATE content_drafts SET payload = ?, save_version = save_version + 1 WHERE id = ?"
).run(payload, draftId);
```
两个安全缺陷叠加：
1. **无归属条件**：WHERE 只有 `id`，没有 `owner_id`。谁能写他人草稿，完全取决于此前是否做过归属校验——方案通篇没提。
2. **TOCTOU**：`proposal.md:12-14` 先 `SELECT` 比对 `current.save_version`，再在 `15-17` 无条件 UPDATE。检查与写入之间是两条独立语句，没有任何原子性保证。

**② 后果**
- **越权写入**：草稿 id 是 UUIDv7（前 48 bit 是毫秒时间戳，见 `migrations/0001_baseline.sql:188` 的 `id TEXT PRIMARY KEY` 与 `src/server/modules/reviews/index.ts:206-211` 的 `parseDraftId`）。UUIDv7 **可预测**，拿到任意一个 draftId 就能覆盖别人的正文。仓库现有代码正是靠显式归属断言挡住这一点：
  - `src/server/modules/reviews/index.ts:657` `if (row.owner_id !== userId) throw err.forbidden('只能保存自己的草稿。')`
  - `src/server/modules/reviews/index.ts:543`（读）、`:575`（改绑）、`:392`（删）三处同款断言。
  方案里这句断言**完全消失**。
- **乐观锁失效（版本控制本身被绕过）**：这正是本方案想解决的问题，却在实现上又丢了。反例：设备 A 与设备 B 同时读到 `save_version = 3`，两个请求都在 `12-14` 通过比对，然后都执行 `15-17`——甲方被覆盖，且 `save_version` 变成 5（跳过 4），**"不会互相覆盖"这条预期（`proposal.md:27`）不成立**。
- 现有实现已用单条 CAS 语句规避，可作对照证据：`src/server/modules/reviews/index.ts:664-670`
  ```sql
  UPDATE content_drafts
     SET payload = ?, save_version = save_version + 1, saved_at = ?
   WHERE id = ? AND save_version = ?
  ```
  并在 `:672-678` 用 `updated.changes === 0` 判冲突。`tests/drafts.test.ts:252-256` 是这条不变量的哨兵断言（注释明写"去掉 WHERE 的 save_version 条件 → 本用例失败"）。

**③ 建议**
- 若本任务确实要落地代码：把 `proposal.md:15-17` 整段替换为上述单条 CAS，并在同一事务内先做 `owner_id` 断言（现状 `index.ts:655-670` 已是这个顺序：先 `readDraftRow` + 归属，再 CAS）。
- 加一条回归测试：**他人草稿 id + 正确 expectedVersion** → 必须 `403 forbidden` 且 payload 未变。现有 `tests/drafts.test.ts` 覆盖了"过期版本不覆盖"，但未见"跨用户写入"用例，建议补。

---

## 二、方案缺"请求去重"，自动保存会放大重放语义问题（P1）

**① 哪部分有问题**
`proposal.md:10` 的请求体只有 `{ draftId, payload, expectedVersion }`，没有 `requestKey`；`proposal.md:20-21` 又要求"每 30 秒自动保存"。

**② 后果**
- 仓库的保存路径是**带幂等键**的：`src/app/api/drafts/route.ts:87-91` 强制 `requestKey`，服务层 `src/server/modules/reviews/index.ts:636-652` 用 `draft_save_receipts` 做"同键同摘要 → 返回原回执、不递增版本"。方案去掉它之后，**网络重试 / 移动端切网重发会被当成新的保存**，在 `save_version` 上留下重复推进，客户端的 `expectedVersion` 与实际版本错位，于是用户会频繁看到"请刷新"的假冲突（`proposal.md:13`、`:24`）。
- 更关键的安全面：幂等表现在是"保存成功才写回执"（`index.ts:680-687`），这保证了重放**不会**绕开 CAS。若后续为了修假冲突而在方案里"放宽 conflicts"，就会把这条不变量拆掉。

**③ 建议**
- 方案必须写明 `requestKey`（客户端为每次**用户可见的保存动作**生成一个，30 秒轮询复用同一个 key 直到内容变化），并明确"重放判定在 CAS 之前"（`index.ts:604-608` 的注释理由原文可引）。
- 明确 30 秒自动保存与手动保存**共用**同一条保存入口，不要另开一条不带幂等键的旁路。

**④ 附带**：新增高频写入口前，先确认频控。`src/server/platform/rate-limit.ts:46-55` 的 `RateLimitAction` 里**没有** `draft_save`，`src/app/api/drafts/**` 全程未见 `enforceRateLimit`。30 秒一次对单用户无害，但若方案被复制到导入/批量场景，就是无上限写库。建议在方案里显式写一句"自动保存按 (user, draft) 节流，服务端以 CAS 冲突率为背压"。

---

## 三、草稿 payload 无尺寸上限：草稿 → 提交是"未过滤的长度通道"（P1）

**① 哪部分有问题**
`proposal.md:16` 的 `payload` 直接入库；全文没有任何长度/结构上限。校验函数 `src/server/modules/reviews/index.ts:311-369`（`parseDraftPayload`）只校验**类型与结构**（title 是 string、body 是数组、每段 markdown 是 string），**没有 maxLength**，`body` 数组本身也没有元素个数上限。

**② 后果**
- 草稿是**长期保存**的（`migrations/0001_baseline.sql:193` `payload TEXT NOT NULL`），保存路径无限流（见二④），`docker/nginx/nginx.conf:54` 只给了 `client_max_body_size 12m`。一个登录用户可以用 12MB × 每 30 秒把 SQLite 主库（web/worker 双进程共享同一个 `.data/app.sqlite`）撑到磁盘打满，是**存储型 DoS / 全站不可写**。
- 与同库其他文本接口的口径**不一致**，说明这是遗漏而非设计：评论有 `COMMENT_BODY_MAX_LENGTH = 20_000`（`src/server/modules/comments/index.ts:61`），媒体有 `MEDIA_LIMITS.maxBytes = 10MB`（`src/server/modules/media/service.ts:34-38`），而**正文草稿反而没有**。
- 连带影响排查与备份：草稿表被放大后，`scripts/diagnose.mjs:30` 的表清单与备份体积同步膨胀。

**③ 建议**
- 在 `parseDraftPayload` 里给 `title` / 每个 `body[].markdown` / `body.length` 加服务端上限（与提交侧 `submit-review.ts` 的正文上限取同一个常量，避免两处漂移），失败走 `err.input` 而非截断（截断会静默吃掉用户内容）。
- 方案里补一句"payload 上限与提交上限同源"作为生效条件。

---

## 四、"冲突时提示刷新、丢弃本地修改"——这是最容易改坏鉴权分支的写法（P1）

**① 哪部分有问题**
`proposal.md:23-24`：
> 冲突时 / 提示用户刷新，丢失本地修改。

**② 后果**
- 后果不在"丢数据"（那是产品问题），而在**实现倾向**：当一个"冲突"要求用户放弃本地内容时，最常见的偷懒修法是"冲突时就强制覆盖/服务端自动接受最新版本"。一旦落地成 `WHERE id = ?`（即回到一里的无 CAS 写法）或"冲突后按服务器版本强制写入"，就同时**破坏了归属校验与版本控制**两条安全边界。当前实现的 `err.conflict`（`index.ts:672-678`）是**拒绝分支**，带 `current_version` 但不写库——这个"失败不改状态"的性质必须保留。
- 另外，方案未说明冲突响应是否回显 `payload`。仓库现在的 `getDraft` 只在**归属校验之后**回显（`index.ts:537-546`，`:543` 断言 owner）。若为"帮助用户合并"而在 409 响应里带回草稿内容，会在冲突路径上**新增一处需鉴权的回显点**，必须复用 `getDraft` 而不是在错误构造里拼。

**③ 建议**
- 方案里把冲突分支写死为"**不写库、不改版本、仅返回 `current_version`**"（对齐 `tests/drafts.test.ts:258-262`：冲突请求不得写回执）。
- 若要做"草稿历史/合并"，明说它走 `getDraft` 的归属校验，不新增回显面。

---

## 五、遗留草稿生命周期：删除草稿会撤回已发布评测（P2，方案未触及但会被"自动保存"触发）

**① 哪部分有问题**
方案只讲保存，未讲草稿在提交后如何与已发布内容解耦。

**② 后果**
- `src/server/modules/reviews/index.ts:394-400` 与 `:422-426`：草稿若已被提交（`content_submissions` 有引用），删除走**软删除**，且 `deleteDraftInTx` 的注释明确"删除草稿只删草稿，不撤回、不影响已提交/已发布评测"。但 `src/app/api/drafts/route.ts:169` 仍在响应里返回 `withdrawn: result.withdrawn`，对上层是不一致的口径——对接方若按 `withdrawn && 就当作已下架` 处理，会误判公开状态。
- 版本控制本身会**增加 `draft_save_receipts` 行数**（每次保存一行，`migrations/0001_baseline.sql:208-218`，主键 `(draft_id, request_key)`，无 TTL）。30 秒自动保存若每次都换新 key，单草稿一天就 ~2880 行。

**③ 建议**
- 回执生命周期要在方案里定：要么"自动保存按 (draftId, 内容摘要) 复用 key"（天然去重），要么明确回执的清理策略（参考 `index.ts:401` 删除草稿时连带删回执）。

---

## 六、并发下的错误信息泄露面（P2）

**① 哪部分有问题**
`proposal.md:13` 的 `throw err.conflict('草稿已被修改，请刷新')` 只有一个文案，没有区分"不存在"、"不是你的"、"类型不对"。

**② 后果**
现有实现刻意做了**差异化且不泄露存在性**的处理，方案把它抹平了：
- `index.ts:656` `row.draft_type !== 'review'` → `err.unavailable`（404 语义）
- `index.ts:657` 归属不符 → `err.forbidden`
- `index.ts:674` 版本不符 → `err.conflict`

若把"不存在/非本人/版本冲突"统一成一句"请刷新"，看似减少了枚举面，实际会让**所有异常都落到 409**，前端无法区分"该重新登录"和"该刷新版本"，用户会反复重试到触发更多写请求。反过来说，若错误地给"非本人"回 `forbidden` 与"不存在"回 `unavailable` 的差异过大，又构成**草稿存在性枚举**。当前实现用 404 掩盖 `draft_type` 差异（`index.ts:656` 直接复用"草稿不存在或已不可访问"文案）是正确取向。

**③ 建议**
方案里补一张"错误码 → 语义 → 是否泄露存在性"的表，明确：**归属不符与不存在都回同一条不可用语义**（避免枚举），版本冲突单独回 409。

---

## 七、跨站写入：Cookie 是 SameSite=Lax，方案未提 CSRF（P2）

**① 哪部分有问题**
`proposal.md` 假设客户端直连 API 保存，未提任何跨站防护。

**② 后果**
- 会话 Cookie 为 `sameSite: 'lax'`（`src/server/platform/session-cookie.ts:97`、`:109`），`Path=/`。Lax 会拦截跨站的 **POST/PUT**，因此当前 `PUT /api/drafts`（`src/app/api/drafts/route.ts:81`）在浏览器口径下是安全的——但这**依赖"保存必须是 PUT 而非 GET"**这一事实。
- 方案若被实现成"自动保存走 `GET /api/drafts?payload=...`"或"用 image beacon / `<img>` 发 GET"，Lax 就放行跨站 GET，**第三方页面就能触发保存**。方案没有明写方法约束，这是可被误实现的空间。

**③ 建议**
- 方案明确一行：**保存只接受 PUT（或 POST）+ `application/json`**，禁止任何 GET 写路径；并保留 `currentContext()` 的服务端会话解析（`route.ts:84`）。

---

## 八、给主评审的落地建议（安全侧最小改动集）

1. **不要按 `proposal.md:15-17` 重写 UPDATE**。现有 `index.ts:664-670` 的 CAS + `:657` 的归属断言已经是正确实现，本任务的真实缺口是**测试**而非代码。
2. 补三条安全回归测试：
   - 跨用户 draftId 保存 → 403，且对方 payload 字节不变；
   - 并发两个同 `expectedVersion` 保存 → 恰好一个成功、一个 409，`save_version` 只 +1；
   - 冲突请求不写 `draft_save_receipts`（`tests/drafts.test.ts:258-262` 已有同款，扩展成跨用户场景）。
3. `parseDraftPayload` 加长度上限（同源常量），并在方案"预期"里把它写成**安全不变量**而不是体验优化。
4. 方案文档补"方法约束（PUT）+ requestKey 幂等 + 冲突分支不改库"三条生效条件，避免后续实现者按草稿版原文落地。

---

### 严重度汇总

| 编号 | 严重度 | 一句话 |
|---|---|---|
| 一 | **P0** | `proposal.md:15-17` 的 UPDATE 无 `owner_id`、无 CAS → 可覆盖他人草稿 + 并发覆盖，版本控制自破 |
| 二 | P1 | 缺 `requestKey` 幂等，30 秒自动保存会制造假冲突/重放歧义；保存路径无频控 |
| 三 | P1 | payload 无长度上限，草稿是长期存储且无限流 → 存储型 DoS |
| 四 | P1 | "冲突即丢弃本地"易被实现成强制覆盖，拆掉 CAS 与归属两条边界；错误响应不得回显他人草稿 |
| 五 | P2 | `draft_save_receipts` 无 TTL，自动保存会线性膨胀；`withdrawn` 响应口径不一致 |
| 六 | P2 | 错误码被抹平成单一 409，既不能指导前端也需防存在性枚举 |
| 七 | P2 | 未约束 HTTP 方法；Lax Cookie 下 GET 写路径会被跨站触发 |
