# 安全评审：n54 数据保留期与自动过期

评审对象：tasks/n54-retention-policy/environment/proposal.md
角色：安全工程师 · 会话 af84bc98-4b8e-4047-86ba-53d1bb6aa653

结论：**不建议按现方案实施。** 方案在当前 schema 上有三条 DDL 级阻塞（已实证），
且其中「删除 operation_records」一项与安全修复单 0042 直接冲突。

## 已实证的逐表行为

以与本仓一致的约束建模（FK ON + 0042 触发器）后实测：

| 表 | 结果 | 原因 |
|---|---|---|
| operation_records | **失败** IntegrityError | 0042 trg_operation_records_no_delete 抬起 ABORT |
| content_drafts（有回执） | **失败** FOREIGN KEY | draft_save_receipts / attachments / 导入批次 ON DELETE RESTRICT |
| content_drafts（无回执） | 成功 | — |
| reports（已处理） | **失败** FOREIGN KEY | report_handling_records ON DELETE RESTRICT |
| reports（未处理/已驳回） | 成功 | — |
| notifications | 成功 | 无引用者（对照组） |

关键放大效应：for 循环无逐表 try/catch，**第一张表 operation_records 抛错即中断整个循环**，
notifications 一条都清不掉 —— 最该清的（PII）没清，方案报「任务失败」。

## 问题清单

### 1. 【严重】与审计不可变安全修复单直接冲突
- 依据：migrations/0042_audit_immutable.sql:17-21；migrations/0001_baseline.sql:411-431
- 0042 立单背景明文写：删除审计行会让已消费的 operation_key 重新可用，构成**伪造通道**（0042:11-12）。
- 建议：operation_records 从本方案**永久排除**；确需缩容走「归档到只读表 + 摘要固化」，绝不 DELETE，
  也不得为了让 DELETE 通过而删掉 0042 的触发器。

### 2. 【严重】for 循环无错误隔离，一表失败则全部不执行
- 依据：proposal.md:20-23；worker 现状 src/worker/scheduled.ts:1-18
- 建议：逐表独立 try/catch + 单表限时；失败不影响其它表；结果写审计与日志。

### 3. 【严重】删除即销毁合规与申诉所需的审计链条
- 依据：migrations/0012_interactions.sql:64-74（report_handling_records）、0001_baseline.sql:413-422
- resolved/dismissed 的举报被删除，等于销毁「我们是怎么处理的」记录；被隐藏内容的作者申诉时无据可查。
- 建议：只清 open 且超期的；resolved/dismissed 整体脱敏保留。

### 4. 【严重】清理与备份保留期不对齐，删除会被备份「复活」
- 依据：src/app/api/admin/backups/route.ts:89（BACKUP_KEEP 默认 14）、scripts/backup.mjs:38,148-153、
  docs/备份与恢复.md:25,69-70
- 每日全库热备、保留 14 份；db 级 DELETE **不会**清理 backups/ 里的副本。
  今天删掉的 400 天前审计记录，仍完整存在于 14 份历史备份中 →「合规有据可依」不成立。
- 建议：保留期 ≥ 备份保留天数；或改备份策略；异地副本明确为人工步骤。

### 5. 【严重】内联 SQL 拼接 + 表名来自对象键
- 依据：proposal.md:22（DELETE FROM ${table}）
- 当前是常量字面量，眼下不可注入；但这是「表名来自数据」的模板，一旦交给后台配置即成为注入面。
- 建议：白名单映射 + 预编译语句；未知键直接抛错。

### 6. 【高】时间单位与口径不匹配
- 依据：proposal.md:21（Date.now() 毫秒浮点）vs created_at INTEGER NOT NULL（0001:196,425；0012:52；0037:293）
- 建议：统一走 nowMs()（src/server/platform/time.ts）并显式 Math.trunc；补「边界前后各一行」回归用例。

### 7. 【高】删除本身不可审计
- 依据：proposal.md:26-27（只有「挂 worker 每日任务」）
- 无 operation_records 留痕、无删除计数、无 cutoff 记录 → 误删无法复盘。
- 建议：每表每次产出 {table, cutoff, candidate_rows, deleted_rows, error}，写 operation_records + 结构化日志。

### 8. 【高】缺少人工刹车与安全阀
- 依据：src/worker/scheduled.ts:45-53（既有 env 刹车范式）
- 建议：DSH_REVIEW_RETENTION_OFF=1 硬刹车；后台开关；首次上线先 dry-run，连续两次跑数一致再真删；单次删除上限 + 告警。

### 9. 【中】PII 的数据主体权利与例外未定义
- 依据：0012_interactions.sql:44（reporter_id）、0037_forum.sql:283-288（recipient_id/actor_id）
- 建议：补数据处理说明（目的、保留期依据、为投诉核查保留的最小例外、异议渠道）。

### 10. 【中】content_drafts 误删他人未发布内容
- 依据：0001_baseline.sql:187-206（草稿是作者唯一的未发布编辑载体）、0018_draft_soft_delete.sql:11（已有软删）
- 建议：走软删/归档，或至少先通知作者；不要首次上线就硬删。

### 11. 【低】「过期」语义未定义
- 建议：明确定义到期基准（created_at vs 最后活跃），否则测试与合规口径无法对齐。

## 若坚持推进的最小改动
1. RETENTION 去掉 operation_records；notifications/reports/content_drafts 分三批上线。
2. 每表独立 worker handler（走既有 tasks 队列），不写在同一个循环里。
3. 表名白名单预编译 + 显式 dry-run。
4. 保留期 ≥ 备份保留期，或改备份策略。
5. 每轮写 operation_records + 结构化日志 + 后台可查的清理报表。
