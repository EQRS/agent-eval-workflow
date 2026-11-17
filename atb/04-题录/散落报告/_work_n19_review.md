# 评审结论：tasks/n19-content-moderation/environment/proposal.md

评审对象共 25 行，技术内容约 15 行（proposal.md:8-25）。**总判定：不可直接实施。**

根本原因一句话：方案在描述一套尚不存在的前置假设（同步审核、失败放行、成本可控），
而本仓库已有一整套**相反约束**的审核链路（不自动放行、异步队列、逐条审计、状态机 CAS）。
方案一旦落地，它不是"新增功能"，而是**推翻已有验收纪律**并在多处产生回归。

---

## P0（阻断级：与现有代码/验收直接冲突，落地即破坏现有保证）

### P0-1 「同步调用审核 API」架空了整个任务队列与状态机

- **哪部分**：§1 审核流程（proposal.md:9-15）—— 提交后同步 call API，判定通过直接 `saveContent`。
- **后果**：
  - 本仓的提交是先落库、再异步入队：提交事务写 `content_submissions`（初始 `pending`），
    并在事务外 `enqueue({taskType:"review_check"})`（src/server/workflows/submit-review.ts:258-305）。
    随后 worker 的 `review_check` 处理器才跑 `runRuleCheck`（src/worker/handlers.ts:105-124）。
    方案的"同步审核"与这条链路是两套并行机制，同一次提交会产生两条审核结论；
    `content_moderation_records` 上的 `UNIQUE (submission_id, operation_key)`（migrations/0001_baseline.sql:239）
    会让两边写入互相打架（幂等命中后，后写的一方静默拿到旧记录 ID，见 moderation/service.ts:231-234）。
  - 状态机是 7 态 CAS（`processing_state` + `state_version`，migrations/0001_baseline.sql:315-317；
    乐观锁在 moderation/service.ts:270-290）。方案的 `saveContent` 完全绕过 `state_version`，
    并发下会让状态与实际上线内容错位。
- **建议**：不要新建"提交内同步审核"路径。把大模型**做成 `contentScreen` 适配器注入现有服务**
  （契约已存在：`ModerationOptions.contentScreen`，moderation/service.ts:159-166），
  由 `runRuleCheck` 编排、由 worker 执行、由 `transition()` 落状态。
  方案里应画出"提交 → enqueue → review_check → contentScreen(LLM) → 状态转移"这条现有链路，而不是另起一条。

### P0-2 §3「API 不可用时先放行并标记待审」直接违反仓库的头号硬规则

- **哪部分**：§3 兜底（proposal.md:20-21）。
- **后果**：这是本仓明文禁止的行为，且已有测试守卫：
  - 规则写在 src/server/modules/moderation/service.ts:9-13："外部内容筛查未配置时**绝不自动放行**、也不认定违规，停在 `manual_review`"；
  - 实现写在同文件 424-451 行：`not_configured / fault / inconclusive` **一律不自动放行**；
  - 回归测试锁死该行为：tests/decide-submission.test.ts:138-143（未配置筛查 → 必须落 `manual_review`，且队列里不得有发布任务）；
  - 部署侧同样声明落 `not_configured → manual_review`"保守，绝不自动放行"（src/worker/main.ts:237-238）。
  - 而且"标记待审"在现有模型里**无对应状态**：`SubmissionStates`（src/packages/common/field-rules.ts:61-70）没有"放行但待审"这一态；
    `publishReview` 只接受 `processing_state === "approved"`（src/server/workflows/publish-review.ts:87-90）。
    方案要么新增状态（迁移 + 全部 CHECK + 前端文案），要么就是在 `manual_review` 之外偷偷放行——后者等于把审核做成装饰。
- **建议**：兜底改为与现有一致的三值语义：`pass / violation / inconclusive`，故障即 `inconclusive → manual_review`
  （moderation/service.ts:359-368 的 `fault` 分支已经就位）。
  如果坚持"降级放行"，必须作为使用者裁定的产品决策，并同步修改 0001 状态 CHECK、`publishReview` 门禁、
  `src/packages/common/status-text.ts` 与 tests/decide-submission.test.ts，不能只在方案里一句带过。

### P0-3 「同步调用」在生产部署形态下不可用，且会放大 web/worker 耦合

- **哪部分**：§1（proposal.md:9）+ §3（proposal.md:20）。
- **后果**：
  - 生产是 web 与 worker 两个独立进程/容器、共用一份 SQLite（WAL）（deploy/dsh-review-worker.service:6-7；
    docker/web.Dockerfile:3-8 声明 web/worker/migrate 共用一份镜像）。审核若在 web 请求线程里同步调用，
    等于把第三方可用性塞进提交请求的关键路径：供应商抖动 → 提交接口超时 → 用户内容丢失
    （`submitReview` 的事务已提交、任务却在事务后才入队，见 submit-review.ts:291-305）。
  - 现有服务的约定恰恰相反：适配器**必须在事务外调用、不得在事务内做网络请求**（moderation/service.ts:163、356-368），
    并且已有"故障即 `fault`"分支。同步方案把这条约定作废。
- **建议**：保持"提交只落库 + 入队"，LLM 调用放在 worker；提交接口只返回 `processingState / taskId`
  （submit-review.ts:307-313 的返回形状），前端轮询或订阅既有 `moderation_result` 通知（src/server/modules/notifications/index.ts:151-168）。
  若业务确实要求"提交时即时反馈"，应做成"同步预判 + 异步终局"，且同步结果不得直接决定 `approved`。

---

## P1（严重：设计缺口会在实现中被发明出错误语义）

### P1-1 错误码不在既有错误词表内

- **哪部分**：`throw err.input("内容不合规")`（proposal.md:13）。
- **后果**：`err.input` 的语义是表单字段校验失败（src/packages/common/errors.ts:79-90），返回 422 并携带 `fields`；
  审核判定不属于输入格式错误。用 `err.input` 表达"内容违规"会导致：
  ① 前端可能把它当字段错误渲染到标题/正文输入框（错位提示）；
  ② docs/API端点与路径.md:17 要求 message 是可直接展示的中文，但审核拒绝需要带**可申诉的理由与依据**，
  而现有 `needs_changes` 路径是通过 `deliverModerationResultNotification` 把原因发给作者的（moderation/service.ts:512-536）。
- **建议**：违规走 `needs_changes` 状态 + 通知（或新增专门的 moderation 错误码），并明确"拒绝理由是否对用户可见、是否含供应商原始文本"。

### P1-2 违规处置没有申诉与人工复核回路

- **哪部分**：§1（proposal.md:9-15）只写了 `throw`，无恢复路径。
- **后果**：本仓对"机器判错了怎么办"已有先例但方案未继承：
  管理员队列与人工改判 `GET /api/admin/submissions`（src/app/api/admin/submissions/route.ts:20-52）、
  `decideByAdmin`（moderation/service.ts:464-539）；申诉表已存在 `plugin_pack_appeals`（migrations/0028_pack_appeals.sql:26）。
  方案的 `throw` 会让被判违规的内容直接消失、作者拿不到理由、也无法申诉——这在评论这类高频内容上是必然的投诉源。
- **建议**：明确定义三件事：①违规是否对作者可见（`needs_changes` + 原因）；②是否支持申诉（复用 pack appeals 或新建）；③误判率统计与人工抽检比例。

### P1-3 未说明审核输入的范围（正文 markdown / 图片 / 链接 / 标题）

- **哪部分**：`callModerationApi({ text: content })`（proposal.md:12）。
- **后果**：现有基础筛查必须自己按 submissionId 读正文，因为 moderation 传给适配器的 snapshot **刻意不含 body**
  （src/worker/content-screen.ts:134-156 的注释与 `loadSnapshotForScreen`）。
  方案里的 `content` 是未定义变量：只审标题？只审正文？审 markdown 源还是渲染后文本？
  - 若审 markdown 源，URL、代码块、引用里的文本会被误伤（本仓评测大量贴命令与日志）；
  - 若不含图片/附件，则"违规图片"完全漏过（本仓有封面与附件链路，submit-review.ts:246-254）；
  - 评测正文可达数万字，超长截断策略缺失 → 尾部违规内容永远不被看到。
- **建议**：逐字段列出审核面（title / body_markdown / 附件 / 外链），给出长度上限与截断/分段策略，以及各类内容的判定动作。

### P1-4 「效率提升 10 倍」无口径、不可验收

- **哪部分**：§预期（proposal.md:23-25）。
- **后果**：无基线（当前日审量？单条耗时？）、无指标定义（效率=吞吐还是时延？）、无准确率/误杀率。
  这种预期无法写成测试或验收脚本，验收时只能"看起来变快了"。
- **建议**：改成可测指标：①进入 `manual_review` 的日均条数变化；②标注集上的召回/误杀率与置信区间；
  ③P95 端到端时延；④供应商故障率与降级占比。并要求给出可回放的评测集（可复用 tests/e2e-slice.test.ts 的端到端口径）。

---

## P2（风险项：合规、成本、运维、可观测）

### P2-1 数据出境与隐私：未说明会把用户内容送到第三方

- **哪部分**：§2 模型选择（proposal.md:17-18）只写"某商业 API，按调用量计费"。
- **后果**：用户评测与评论是**署名公开作品**（src/server/modules/forum/boards.ts:100 明确"帖子正文是作者的署名作品"），
  方案把全文送第三方模型却不提：供应商是否留存/训练、是否跨境、是否需要在用户协议里告知。
  本仓别处对"不对外上报"有明确纪律（docs/安全审查-第一集整合包.md:135-139：只在用户点击时才上报）。
  另外 `content_moderation_records.evidence` 是**落库长期留存**的 JSON（migrations/0001_baseline.sql:237）：
  若把供应商原始响应（可能含被判违规的原文片段）写进 evidence，等于把违规文本长期留在库里。
- **建议**：补隐私章节：数据最小化（只送必要字段）、供应商数据保留条款、
  evidence 只存判定码与摘要/哈希不存原文片段、必要时脱敏。

### P2-2 成本与配额完全不可控，且没有幂等/缓存设计

- **哪部分**：§2（proposal.md:18）"按调用量计费"。
- **后果**：本仓是"总量小、重试多"的场景：worker 任务有重试（故障码分类见 src/worker/handlers.ts:356），
  审核调用失败重试每次都计费；同一提交可能被 `review_check` 重复跑（幂等键 `review_check:<submissionId>` 只在入队侧去重，submit-review.ts:297）。
  没有单用户频控 → 恶意用户刷提交即可刷高账单。本仓频控"按用户计、不按 IP"（docs/HANDOFF.md:122），方案没接这条。
- **建议**：写明单条与单日预算上限、超限降级到 `manual_review`（而不是放行）、
  以 `submissionId` 为键缓存判定结果、失败重试次数上限与退避策略。

### P2-3 运维与可观测缺位：故障时谁来发现

- **哪部分**：§3 兜底（proposal.md:20-21）只讲"放行并标记"。
- **后果**：现有可观测面是 worker 心跳 + `/api/health`，且 health **刻意不含 worker 心跳**（docker/web.Dockerfile:74-77）。
  若审核在 web 进程里同步跑，供应商整体故障不会体现在任何健康信号上，只表现为提交变慢/失败；
  `fault` 记录虽然落库（moderation/service.ts:425-445），却没有"故障率超阈值告警"的设计。
- **建议**：补审核调用成功率/降级率埋点 + 阈值告警；故障态在后台待办里可见（可复用 `GET /api/admin/submissions?state=manual_review`，src/app/api/admin/submissions/route.ts:27）。

### P2-4 未定义"审核通过"的可解释证据与审计要求

- **哪部分**：`verdict.blocked` 布尔（proposal.md:13）。
- **后果**：现有审计模型要求每次处理留一条记录，含 `stage / result / actor_kind / reason / evidence`，
  且 system 身份必须带 `task_id`（migrations/0001_baseline.sql:227-245；写入前在 moderation/service.ts:217-229 校验）。
  方案的 `blocked: boolean` 丢失了"谁、依据什么、什么时候"三要素，与 `result` 的五值枚举
  （`pass / needs_changes / manual_review / fault / ineligible`，src/packages/common/field-rules.ts:73）也无法对齐——
  `blocked` 该映射到 `needs_changes` 还是新值，方案没说。
- **建议**：适配器返回值收敛到既有三值 `pass/violation/inconclusive`（src/worker/content-screen.ts:38），
  并附带 `reason` 与 `fields`（可解释依据），由服务端写审计记录，而不是自带一套布尔。

### P2-5 供应商信息未落地，无法做稳定性与切换评估

- **哪部分**：§2（proposal.md:17-18）"某商业 API"。
- **后果**：无法评估 SLA、单价、限流、中文效果、长文本支持、阈值可配性。
  更关键的是——现有代码里只有一个本地占位筛查，且明确写着"生产部署必须替换成真实内容审核供应商"（src/worker/content-screen.ts:18）。
  方案正是要补这个空缺，却没给出候选与对比。
- **建议**：给出 2-3 个候选对比表（单价/限流/中文效果/数据条款/SLA），并明确开关方式
  （现有 `DSH_REVIEW_CONTENT_SCREEN` 只有 `basic` 与未设两态，docker/.env.example:48；需扩展取值与装配点 src/worker/handlers.ts:117-122）。

---

## P3（文档与流程）

### P3-1 方案未说明与既有配置开关、迁移的关系

- **哪部分**：全文未提配置项、灰度、回滚。
- **后果**：现有开关是 `DSH_REVIEW_CONTENT_SCREEN=basic`（src/worker/content-screen.ts:127-129）。
  接入 LLM 后需要新增取值（如 `llm`）、按用户/内容类型灰度、以及"关掉就回落到 manual_review"的回滚路径。
  若要新增状态（见 P0-2），还需新迁移 + 更新 src/server/platform/migrate.ts 的表清单（:153-154 已登记这两张表）。
- **建议**：补"配置/灰度/回滚"一节，并把迁移编号、受影响表、CHECK 变更列为交付物。

### P3-2 未提及对既有测试与验收的影响

- **哪部分**：全文。
- **后果**：tests/decide-submission.test.ts:138-143 明确断言"未配置筛查 → 不得自动通过、不得有发布任务"。
  方案的兜底语义（放行）与该断言直接相反；实施时不改测试就会红，改了测试就等于悄悄放宽纪律。
- **建议**：列出需要变更/新增的测试清单与理由，明确"哪条纪律被有意改变、由谁批准"。

---

## 附：评审中确认相符的部分

- 人工审核确实已存在且是现状主路径（`GET /api/admin/submissions` 默认拉 `manual_review`，src/app/api/admin/submissions/route.ts:27），
  所以"人审跟不上"这个背景陈述是**成立**的。
- 仓库确实只有占位筛查、没有真实供应商（src/worker/content-screen.ts:1-21），"要接大模型"的方向本身是**对的**。
  问题全部出在接入方式（同步、放行兜底、无口径），而非方向上。
