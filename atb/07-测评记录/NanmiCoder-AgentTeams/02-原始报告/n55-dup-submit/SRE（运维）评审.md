# n55-dup-submit SRE（运维）评审

> t45 · e3-fixer · 2026-11-17

---

n55-dup-submit SRE 视角评审完成。落盘：atb/tasks/n55-dup-submit/review-sre.md（118 行，只读评审，未改源码）。

结论：方案要解决的问题在本仓库已经解决了（落库的 request_key 幂等：draft_save_receipts / content_submissions），而它提的两个机制——前端按钮态、进程内 Set——一个是 UX、一个在多进程+重启下天然不成立。本仓正是双容器+restart 常态部署。

摘要前 5 条：
1.【严重】§2 的进程内 Set 失效：compose.yaml:44-76（web）与 :78-94（worker）是两个容器、:68,:93 restart: unless-stopped、deploy/dsh-review-worker.service:17 独立进程。重启即清空；且它只解决「同一时刻」，解决不了「上一次已提交完用户又点一次」——后者才是重复创建的真来源。
2.【严重】仓库已有跨进程跨重启且带摘要校验的幂等：0001_baseline.sql:208-218（draft_save_receipts PRIMARY KEY (draft_id, request_key) + CHECK(result_version=expected_version+1)）、:300-332（content_submissions request_key/request_digest + UNIQUE(submitted_by, request_key) :321）；实现 reviews/index.ts:601-691（先查回执→同键同摘要返回原结果→同键不同摘要 keyReused→CAS→写回执）、submit-review.ts:148-151；request-id.ts:6-15 明写三键边界。全仓 48 个文件出现 requestKey/request_key。
3.【严重】§1 按钮禁用不构成防重，且失败语义会诱发重复：folder-actions.tsx:44-47 的既有做法是失败不恢复 busy；按方案终于 finally 恢复按钮 → 上一次可能已在服务端成功，用户再点就是真重复。
4.【高】§2 把重放误报成 err.conflict → 计入 4xx 错误率与告警，淹没真正的冲突。仓库对同类语义是返回原结果（reviews/index.ts:651 replayed:true、operations/service.ts:115-133）。
5.【中】无「重复提交发生了多少次」的可观测；且日志驱动 json-file max-size 10m/max-file 3 会轮转（compose.yaml:70-74,95-99），所以指标必须落库或计数而不是只靠日志。

完整清单 S-01…S-07，另含：无回滚/共存策略（要替换的是一批生产在用的通路）；inFlight 只用 userId 作键会把用户不同操作互相阻塞（对照 rate-limit.ts:46-55 的按动作独立额度）。

建议 5 条：删除 §2、不引入进程内状态；前端每次用户意图生成 requestKey 并重试沿用；结果返回原资源 + replayed:true 不加 409；加 request_replayed 日志/计数作为验收口径；缺 request_key 通路的接口补唯一约束而不是内存锁。

前提核实：仓库不存在名为 inFlight 的实现（grep 全仓无此标识）；而方案完全没提到的 request_key 幂等机制真实存在且被 48 个文件使用——这是最关键的前提遗漏。