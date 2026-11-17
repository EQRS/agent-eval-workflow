# n42-mlps SRE（运维）评审

> t36 · n46-sre · 2026-11-17

---

N42（等保二级合规改造）SRE 评审完成：结论「有阻塞项（4 P0 + 3 P1），不建议按现行方案上线；方案重复建设了已有的不可变审计表，且新表比现有的更不合规」。报告：_work/评审-N42-等保二级合规-SRE.md

P0-1 §1 新建 audit_log = 开第二套账且无防篡改。项目唯一审计载体是 operation_records（0001_baseline.sql:411-431，字段完整且带 UNIQUE(action_type, operation_key)），并已有 0042_audit_immutable.sql:17-27 的 BEFORE UPDATE/DELETE 触发器（该迁移文件头 0042:5-12 写明没有它"删除审计行即可让 operation_key 重新可用"）。新表无触发器 → 谁都能 DELETE/UPDATE，等保"审计记录免于未预期删除/修改/覆盖"这一条新表过不了；两套账必然漂移；且新表 user_id 无外键、无 actor_kind，系统动作（自动发现 discovery.ts:274、任务重试）无法表达。
P0-2 §2"所有写操作都记一条"把写延迟钉死：better-sqlite3 同步驱动 + 单连接（db.ts:1-11,66-74）+ synchronous=FULL（db.ts:54），方案给的是事务外单独 INSERT → 多一次独立提交/fsync；放进事务则审计会跟业务一起回滚（方案未声明）。且 operation_records 无保留策略且触发器禁止删除 → 单向增长，与媒体共用 /app/.data（compose.yaml:65,92），health.ts:50,94 阈值仅 200MB。
P0-3 §4"保留 6 个月"与现有机制全面冲突：① 备份只有 14 天（dsh-review-backup.service:26 --keep 14、compose.yaml:62），库损坏按 restore-backup.mjs:102-111 恢复的是 ≤14 天前，6 个月审计一起没；② 0042 的 BEFORE DELETE 触发器会让任何"清理超 6 个月"的作业直接 ABORT，"保留 6 个月"在当前实现里无法兑现也无法轮转；③ 容器日志 json-file 10m×3（compose.yaml:70-74,95-99），按 server.mjs 每请求一行，落 stdout 连几小时都保不住；④ 没有任何存放位置/备份频率/恢复演练/到期销毁/访问控制的设计。
P0-4 §3"管理员操作二次密码确认"在本架构里不存在可确认的东西：全站 GitHub OAuth、无 password_hash。实现者只能自建密码体系（新增凭据面）、写无安全价值的确认弹窗、或要 GitHub 密码（绝对不能做）。应改为短时效 reauth ticket（10 分钟 TTL、绑 session），并明确高危操作清单与审计。

P1-5 审计读取面与访问控制未定义：diagnose.mjs:156-171 只列 background_tasks 的 recent_problems，完全不展示 operation_records；后台无审计查询页。建议只读查询 + 读取本身写审计 + 导出限流。
P1-6 ip 字段：来源不可靠（client-ip.mjs:54-71 只有 TRUST_PROXY=1 才读头；worker 无 HTTP 拿不到 IP），三种语义混在一列无法追溯；同一 IP 已在访问日志（server.mjs:71-75）存过，审计再存 6 个月 = 多一份最小化个人信息副本。关键：operation_records.request_reference（0001:423）本是为此留的字段而方案未用——request_id 链路现成（server.mjs:53-60 + request-id.ts:55-57），审计只需记 request_reference 就能与含 IP 的访问日志对上。
P1-7 迁移编号与部署顺序：仓库当前到 0041（0042 已占用、0027/0035 空号），migrate.ts:51-55 按文件名排序且无重复编号检测；compose.yaml:27-42 的 migrate 前置容器只在 compose 生效，systemd 下 worker.service:8 仅 After=，没有任何"先迁移"保证。建议写死文件名 + 上线流程（备份→停 worker→migrate→起）。

另附 8 项上线验收清单（单一审计源/防篡改生效/同事务/6 个月可兑现/读取受控/不拖慢写入/关联可追溯/保留期可配）。