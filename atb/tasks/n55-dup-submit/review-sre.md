# 评审：表单防重复提交 · SRE（运维）视角

> 评审对象：`atb/tasks/n55-dup-submit/environment/proposal.md`（32 行）
> 评审人身份：**SRE / 运维**（只看多进程/重启语义、可观测性、运维可操作性、幂等是否跨实例成立；不评前端交互细节）
> 仓库根：`dsh-plugin-reviews-new`（`src/`、`migrations/` 至 0042、`docker/`、`deploy/`）
> 结论：**方案要解决的问题在本仓库已经解决了，而且用的是它给不出的那种解法**——
> 落库的 `request_key` 幂等（`draft_save_receipts` / `content_submissions`）。§2 的进程内 `Set` 在多进程 + 重启面前**天然不成立**，
> 而本仓的部署形态恰恰是「两个容器 + 宿主 systemd + 会滚动重建」。

---

## 摘要（前 5 条最重要）

1. 【严重】**§2 的进程内 `new Set()` 在多进程/重启下失效——而本仓正是多进程部署。**
   依据：`proposal.md:22-28`；`docker/compose.yaml:44-76`（`web` 服务）与 `:78-94`（`worker` 服务）是**两个容器**；
   `deploy/dsh-review-worker.service:17`（worker 走 `tsx src/worker/main.ts`，独立进程）；
   `docker/compose.yaml:68,93`（两者的 `restart: unless-stopped`——重启是常态，不是异常路径）。
   后果：① 双击落在两个不同的 Next 请求处理上下文里本来就未必同进程（容器重建后更是新进程）；
   ② **重启即清空**：部署窗口（`docker compose up -d`）里的双击，Set 一清就双双放行；
   ③ 更根本地：`inFlight` 只解决「同一时刻」，解决不了「上一次已经提交完、用户又点了一次」——而后者才是重复创建的真正来源。
   建议：**删除 §2**，改用仓库已有的落库幂等（见摘要 2）。

2. 【严重】仓库已有跨进程、跨重启、带摘要校验的幂等机制，方案是在重造一个更弱的。
   依据：`migrations/0001_baseline.sql:208-218`（`draft_save_receipts
  PRIMARY KEY (draft_id, request_key)` + `CHECK (result_version = expected_version + 1)`）；
   `0001_baseline.sql:300-332`（`content_submissions` 的 `request_key` + `request_digest` + `UNIQUE (submitted_by, request_key)`，`:321`）；
   实现：`src/server/modules/reviews/index.ts:601-691`（`saveDraft`：先查回执 → 同键同摘要返回原结果 `replayed: true`，同键不同摘要 `err.keyReused()` → 再 CAS 更新 → 成功才写回执）；
   `src/server/workflows/submit-review.ts:148-151`（提交侧同键先查既有提交，返回原结果）；
   语义说明：`src/server/platform/request-id.ts:1-15`（明确区分 `request_id` / `request_key` / `operation_key` 三个键的用途）。
   后果：用户双击 → 两次请求带**同一个 `requestKey`** → 第二次拿到原结果而不是新建；这在多进程、重启后都成立，因为判据在库里。
   建议：把「防重复提交」实现为「前端为一次用户意图生成一个 `requestKey` 并在重试时沿用」，完全复用上面的通路。

3. 【严重】§1「禁用按钮」不是防重语义：它只覆盖「同一次渲染里的连点」，且会把用户卡在提交态。
   依据：`proposal.md:11-18`（`setSubmitting(true)` … `await submit(); setSubmitting(false)`）；
   对照本仓前端组件的既有写法：`src/components/folder-actions.tsx:31-48`（`busy` 状态在 **catch 里只 setError、不 setBusy(false)**，即失败后按钮保持禁用，直到下一次成功）；
   `src/components/admin-correction-panel.tsx:40` 等也是 `busy` 系状态。
   后果（运维视角）：① 网络抖动导致 `await submit()` 抛错 → 若按方案 `finally` 恢复按钮，用户可以再点一次，此时**上一次可能已经在服务端成功**，于是产生真实重复；
   ② 若按仓库既有写法失败不恢复，用户被卡住，只能刷新——而刷新在没有 `requestKey` 的情况下正是「重复提交」的另一个来源。
   建议：前端保留按钮态只是 UX，**真正的防重必须由 `requestKey` 承担**；失败时明确提示「已提交，请查看」而不是简单放开按钮。

4. 【高】§2 的 `throw err.conflict('请勿重复提交')` 把「用户自己的第二次点击」误报成冲突错误，运维侧看到的是噪音。
   依据：`proposal.md:26`；仓库对同类语义的处理是**返回原结果**而不是报错：`reviews/index.ts:651`（`replayed: true`）、`submit-review.ts:148-151`。
   后果：告警系统会把这类 409 计入错误率；排查真实冲突（版本不一致、越权）时被淹没。这是典型的「用一个新错误码掩盖了一个已解决的问题」。
   建议：重复提交返回首次结果（HTTP 200/201 + 原资源），不加新错误码。

5. 【中】没有任何「重复提交发生了多少次」的可观测——而这正是判断方案是否有效的唯一依据。
   依据：`proposal.md:31-32`（预期只有一句）；本仓可观测手段：`server.mjs:62-82`（每请求一行含 `request_id/status/duration_ms`）、
   `reviews/index.ts:651` 的 `replayed` 标志、`operations/service.ts:115-133` 的 `replayed: true` 同键重放。
   后果：改造上线后无法回答「重复提交还发生过吗」；`inFlight` 这类进程内状态**连日志都不会留**（方案没有打日志）。
   建议：把「重放次数」做成指标（读回执命中时 `log.info('request_replayed', { action, request_key })`），
   并在发布后用 `grep request_replayed` 与 4xx 计数对比，作为验收口径。

---

## 完整问题清单（按严重度）

### S-01 【严重】进程内 `Set` 在多容器 / 重启 / 多实例下失效
- 现场：`proposal.md:22-28`。
- 依据：`docker/compose.yaml:44-76,78-94`（web / worker 两容器）、`:68,:93`（`restart: unless-stopped`）；`deploy/dsh-review-worker.service:17`；
  `deploy/dsh-review-web.service`（宿主 systemd 起的另一个 web 形态）。
- 后果：见摘要 1。补充：即使只有一个 web 容器，Next 的并行请求也可能落在不同的事件循环任务上——`Set` 只能挡住**真正同时**的那一刻。
- 建议：删除；改用落库幂等。

### S-02 【严重】仓库已有更强的幂等：`draft_save_receipts` / `content_submissions`
- 现场：`proposal.md` 全文未提及 `request_key`。
- 依据：`0001_baseline.sql:208-218,300-332`；`reviews/index.ts:601-691`；`submit-review.ts:148-151`；`request-id.ts:6-15`；全仓 **48 个文件**出现 `requestKey/request_key`（grep 统计）。
- 后果：重造机制 → 两套幂等并存，将来漂移。
- 建议：统一到 `request_key`；新增写接口按同一范式（先查回执 → 比对摘要 → 写）。

### S-03 【严重】§1 的按钮禁用不构成防重，且失败语义会诱发重复
- 现场：`proposal.md:11-18`。
- 依据：`src/components/folder-actions.tsx:44-47`（失败只置 error、不恢复 busy 的既有做法）、`:50-65`（删除确认的 busy 处理）。
- 后果：见摘要 3。
- 建议：按钮态只做 UX；防重交给 `requestKey`；失败提示改为「已提交/请稍后查看」。

### S-04 【高】把重放误报成 `conflict`，污染错误率与告警
- 现场：`proposal.md:26`。
- 依据：`reviews/index.ts:651`（`replayed: true`）；`operations/service.ts:115-133`（同键重放返回原记录）；`submit-review.ts:148-151`。
- 后果：见摘要 4。
- 建议：返回原结果，不新增 409。

### S-05 【中】缺「重放/重复提交」可观测指标
- 现场：`proposal.md:31-32`。
- 依据：`server.mjs:62-82`（访问日志字段）；`docker/compose.yaml:70-74,95-99`（日志驱动 `json-file`，`max-size 10m`、`max-file 3`——**日志会轮转，指标必须落数据库或计数**）。
- 后果：无法验收，也无法在事故里回答「这次重复创建是不是防重失效」。
- 建议：`request_replayed` 日志 + 按日计数；验收看 4xx 与重放次数的比例。

### S-06 【中】方案没有回滚/共存策略，而它要替换的是一批生产在用的通路
- 现场：`proposal.md` 全文无「如何与现有 `requestKey` 机制共存」。
- 依据：`request-id.ts:6-15`（三个键的边界）；`reviews/index.ts:601-691`（保存草稿的完整幂等通路）；`content/index.ts` 等 48 个文件涉及 requestKey。
- 后果：如果按方案「后端加锁」另起一套，运维要同时监控两套语义；出问题时无法判断是哪个机制没生效。
- 建议：方案里明确「本改造只新增/复用 `request_key`，不引入进程内状态」。

### S-07 【低】§2 的 `inFlight` 键只用 `userId`，会把用户的不同操作互相阻塞
- 现场：`proposal.md:24-28`（`inFlight.has(userId)`）。
- 依据：仓库写接口是按「动作」分别频控的：`src/server/platform/rate-limit.ts:46-55,68-134`（9 个动作各自独立额度）。
- 后果：同一用户在提交评测的同时保存草稿（或另一个表单），会被误判为「请勿重复提交」。
- 建议：键至少是 `(userId, action, targetId)`；但正确的解法仍然是不用进程内状态。

---

## 结论与建议

**方案是「无用功 + 倒退」**：`request_key` 幂等（`draft_save_receipts` / `content_submissions`）已经在生产里跑了 **48 个文件**，
而方案提出的两个机制——前端按钮态、进程内 Set——一个只是 UX、一个在容器化部署里天然不成立。

**建议**：
1. 删除 §2；**不引入任何进程内状态**作为正确性保障。
2. 前端为每次「用户意图」生成 `requestKey`（`crypto.randomUUID()` 或仓库 `newRequestId`），重试沿用同一个；后端沿用既有通路：
   先查回执 → 同键同摘要返回原结果 → 同键不同摘要 `key_reused` → 再执行。
3. 结果返回「原资源 + `replayed: true`」，不新增 409。
4. 增加 `request_replayed` 日志/计数，作为验收口径（对照访问日志里的 4xx 比例）。
5. 若某些写接口**没有** `request_key` 通路（需逐一核对，见 S-02 的文件清单），补的是那张表的唯一约束，而不是内存锁。

> 评审人声明：本报告只覆盖运维/多进程语义/可观测/运维可操作性视角；前端交互细节与业务语义由其他视角负责。

> 前提核实：方案 §1 的 React 写法与 §2 的 `Set` 都是通用写法，仓库中**不存在**名为 `inFlight` 的实现（grep 全仓无此标识）；
> 而方案**没有提到**的 `request_key` 幂等机制真实存在且被 48 个文件使用——这是本方案最关键的前提遗漏。
