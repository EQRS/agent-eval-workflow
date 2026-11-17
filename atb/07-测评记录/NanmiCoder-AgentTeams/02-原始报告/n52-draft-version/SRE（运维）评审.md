# n52-draft-version SRE（运维）评审

> t39 · e3-investigator · 2026-11-17

---

# n52-draft-version SRE（运维）评审

评审对象：atb/tasks/n52-draft-version/environment/proposal.md（28 行）
已逐行核实 migrations/0001_baseline.sql、src/server/modules/reviews/index.ts、
src/packages/common/autosave.ts、src/components/review-editor.tsx。

**总判：这个功能本仓已经完整实现了，而且比方案更严谨。方案是一份更弱的复刻。**
现有实现：reviews/index.ts:601-690 的 saveDraft（回执幂等 + 原子 CAS + 归属校验），
前端 review-editor.tsx 的自动保存（可配置间隔、停笔静默期、失败不打断写作），
表结构 content_drafts.save_version（0001:195）+ draft_save_receipts（0001:300-332）。

## 最重要 5 条

### S1【阻塞】方案的核心 UPDATE 缺 WHERE 版本条件 —— 存在 check-then-act 竞态窗口
proposal:12-17：先 `if (expectedVersion !== current.save_version) throw`，
再 `UPDATE ... WHERE id = ?`（**没有** `AND save_version = ?`）。
**代码事实**：既有实现是把版本条件放进 WHERE 做**原子**比较——
reviews/index.ts:664-670 `UPDATE ... WHERE id = ? AND save_version = ?`，
:672-678 用 `updated.changes === 0` 判冲突，并在 :662-663 注释说明
「比较的是**提交时请求方所见的版本**，保证并发下只有一个请求能推进」。
② 后果：方案的 SELECT 与 UPDATE 之间有窗口。两台设备在同一毫秒提交，
两边都能通过 check，随后**两次 UPDATE 都成功** → 后写覆盖前写，
正是 proposal:3-4 要解决的「互相覆盖」。方案号称解决却没解决，且更难发现
（因为代码里明明写了版本检查，review 时容易被当成已覆盖）。
③ 建议：**照抄 reviews/index.ts:664-678**，版本比较必须进 WHERE，用 `changes === 0` 判冲突。
**不要自己做一遍**——现成实现已带注释解释每个顺序的理由。

### S2【阻塞】缺幂等回执 —— 自动保存重试会被误判成「冲突」
proposal:10-17 只带 `expectedVersion`，没有 requestKey。
**代码事实**：既有实现有专门的 `draft_save_receipts`（0001:300-332 附近的回执结构与
reviews/index.ts:636-652 的读取逻辑），并在 :604-610 注释写下关键顺序：
「**重放必须在 CAS 之前判定**，否则带着旧 expectedVersion 的重放会被误判成 version_conflict，
客户端重试同一请求就会看到假冲突」。
② 后果：proposal:21 每 30 秒自动保存 + 网络抖动时客户端重试，
第一次已成功但响应丢失 → 重试带着**旧** expectedVersion → 方案判 `save_version` 已变 →
抛 `err.conflict('草稿已被修改，请刷新')`。用户看到的是**假的冲突提示**，
然后按 proposal:24 去刷新，**丢掉本地修改**（proposal:24 正是这么设计的）。
这是**由重试机制本身**制造的数据丢失，运维上会表现为「莫名丢稿」的零星报障，极难复现。
③ 建议：必须带 `requestKey` + 回执表，且**先查回执再做 CAS**（顺序不能反）。

### S3【阻塞】30 秒写死，与既有「可配置 + 0=关闭」的偏好冲突
proposal:21「编辑器每 30 秒自动保存一次」。
**代码事实**：自动保存间隔是**每用户偏好**（迁移 0030，src/packages/common/autosave.ts:9）：
`AUTOSAVE_SECONDS_OPTIONS = [0, 3, 5, 10, 30]`，**:8 与 :11-12 明确 0 = 关闭、默认 5 秒**；
review-editor.tsx:211-212,363 按 `autosaveMs <= 0` 直接不排定时器。
② 后果：若按方案写死 30 秒，会把「关闭自动保存」的用户强行改回自动保存；
而 30 秒也是最差档（写放大最大、冲突窗口最长）。
③ 建议：沿用既有偏好；**不要**新增第二套间隔常量（autosave.ts:1-5 已警示
「白名单两处漂移的后果是设置页能选、保存时被拒」）。

### S4【高】proposal:24「提示刷新，丢失本地修改」是明确的用户体验倒退
**代码事实**：既有实现刻意**不丢**用户输入——reviews/index.ts:673-677 冲突时
「不写回执、不改 payload，用户输入由调用方保留」；
前端 review-editor.tsx:355-357 也写明「失败不弹 ErrorNotice（自动保存不该打断写作），
只在状态条上标「自动保存失败」」，:680-681 注释阐明理由。
② 后果：方案把「保留用户输入」改成「丢失本地修改」，等于用**数据丢失**换一致性——
而这恰好是多端编辑场景里用户最不能接受的失败模式；
运维上它会变成一类无法复现的投诉（用户说不清什么时候丢的）。
③ 建议：冲突时**保留本地内容**，提示「已在别处更新，请选择保留哪一份」；
至少不要在执行层面直接丢弃。

### S5【高】方案漏掉草稿软删与归属校验，会写出「已删除草稿被复活」或越权保存
proposal:15-17 的 UPDATE 只有 `WHERE id = ?`。
**代码事实**：既有实现有两道方案完全没提的校验：
  - **归属**：reviews/index.ts:657 `if (row.owner_id !== userId) throw err.forbidden`；方案没有，
    任何登录用户拿到 draftId 就能改别人的草稿；
  - **软删**：0018_draft_soft_delete.sql:10「列表/详情/保存统一过滤 deleted_at IS NULL，
    删除后不可再读写」，reviews/index.ts:585-588 显式判 `state.deleted_at !== null` 即失败；
    方案不判，会对已提交/已软删的草稿继续 UPDATE payload。
② 后果：前者是**越权写**（可直接用于污染他人稿件）；
后者会让已提交的草稿在提交后被继续改写，造成「提交内容与草稿不一致」，
审计上无法解释（0002 的修订快照不可变，但草稿会被改）。
③ 建议：照抄 :654-657 与 :585-588 的两道校验；UPDATE 的 WHERE 再加 `AND deleted_at IS NULL`。

## 完整清单（其余）

### S6【中】方案没有事务，两次语句之间可被并发插入
proposal:12-17 是独立的 check + UPDATE。既有实现整个 `saveDraft` 在 `withTransaction` 内
（reviews/index.ts:627），回执与 CAS 同事务。
② 后果：无事务时「回执写成功但更新失败」或反之都会留下不可解释的状态；
且 SQLite 下长时间持锁会影响其它写入（busy_timeout 5000ms，db.ts:56）。
③ 建议：与既有实现一致，包进 withTransaction。

### S7【中】未说明冲突的可观测性，运维无法定位「丢稿」投诉
**代码事实**：既有错误 `err.conflict` 会带 `context`（reviews/index.ts:674-677 给出
`current_version` / `expected_version`），经 errors-http 映射成 409 且 context 可公开。
② 后果：方案只抛一条字符串消息，排障时无法区分「真冲突」与「假冲突（S2 的重试）」——
而这恰恰是丢稿投诉里最关键的分叉点。
③ 建议：冲突与重试都要带版本号 context，并在访问日志/审计里留痕（server.mjs 已有 request_id 串联）。

### S8【中】方案未评估自动保存对数据库的写放大
proposal:21 每 30 秒 × 每个在线编辑者。
**代码事实**：db.ts:53-56 `journal_mode = WAL`、`synchronous = FULL` ——
**每次提交都 fsync**。
② 后果：N 个并发编辑者 → 每 30 秒 N 次 fsync；若编辑器同时打开多个标签页还会翻倍。
在低配 VPS 上这是可感知的 IO 压力；而默认 5 秒的既有实现若真开满，压力是 30 秒版的 6 倍——
说明这个参数需要**与磁盘能力挂钩**，不能拍脑袋。
③ 建议：保留可配置间隔（S3），并把默认值当作性能参数在文档里说明依据。

### S9【低】未说明 payload 大小上限
proposal:16 直接 UPDATE payload（`content_drafts.payload` 是 json_valid 的 TEXT，0001:193，无长度 CHECK）。
② 后果：大 payload 每次自动保存都全量写盘（无增量），叠加 S8 的 fsync 会明显变慢；
且无上限时一次超大保存可长时间持锁。
③ 建议：在服务端设 payload 上限（既有 parseDraftPayload 已有结构校验，可顺势加上限）。

### S10【低】验收无口径
proposal:26-28 只有两句定性描述。
建议改成可核查的：并发两个 save 只有**一个**成功（另一个 409 且本地内容不丢）、
重试同一 requestKey 返回原结果不递增版本、关闭自动保存的用户不再产生写入。

## 一句结论
方案要解决的问题**已经解决了**：reviews/index.ts:601-690 的 saveDraft 具备
「回执幂等（重放先于 CAS）+ 原子 CAS + 归属校验 + 软删过滤 + 冲突保留本地内容」，
前端也已有可配置的自动保存与不打断写作的失败处理。
方案的四点里，**核心 UPDATE 少了 WHERE 版本条件（竞态）、缺幂等回执（假冲突→真丢稿）、
写死 30 秒（覆盖用户偏好）、冲突即丢本地修改（UX 倒退）**，
每一条都是把已正确的东西改差。建议**直接复用既有实现**，不要新建一套。