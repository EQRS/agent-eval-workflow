# N-46 评审报告 · NanmiCoder T-安全

> 测评：NanmiCoder Agent Teams `@nanmicoder/dsh-agent-teams@0.1.30`
> 成员：e3-investigator · 任务：t4 · 状态：completed
> 日期：2026-11-17

---

# N-46 安全视角评审（安全工程师）

评审对象：atb/tasks/n46-ownership-dispute/environment/proposal.md（30 行）
基线核实：plugin_ownerships 真实存在于 migrations/0040_user_roles.sql:82-120；认领实现 src/server/modules/roles/ownership.ts。

## 摘要：最严重的 5 条

1. **[阻塞] 没有"旧归属撤销"状态语义，必然撞唯一约束 —— 且方案会诱导实现者改用 UPDATE 覆盖**
   proposal:26「管理员查看证据后，直接改 plugin_ownerships 的归属人」。
   后果：0040:119-120 的 uq_plugin_ownerships_verified 只允许一个 verified，直接改会 SQLITE_CONSTRAINT；更危险的是实现者会去"改 user_id"（UPDATE 而非状态迁移），而 0040:86-95 的 source/status/reviewer_id 约束与"谁在何时复核"的信息会被就地抹掉。另 ownership.ts:245-249、:329-332 已有"taken"前置检查，方案的"直接改"绕开了它。
   建议：不要在 disputes 里改归属；改为调用既有 reviewClaim 语义的"转让"用例：旧行 status→rejected(claimed_note=转让理由)，同一事务内插新行 verified，写 reviewer_id/reviewed_at。

2. **[阻塞] 转让归属 = 转让"开发者"身份门槛，方案对此零讨论**
   proposal 全篇没有提到：plugin_ownerships 的 verified 是开发者身份的**资格判定依据**。
   后果：src/server/modules/roles/service.ts:443-453 与 :475 用 `COUNT(*) ... status='verified'` 决定能否申请开发者；users/[userId] 后台还能直接授予身份。任何人只要能推动一次"归属纠正"，就顺带拿到开发者身份及其后续权限。恶意者可用争议把作者挤下去。
   建议：方案必须声明归属转让对 user_roles('developer') 的连带处理（是否随之撤销、是否需要重新审），并把这个决定写进 criteria 之外的设计文档里。

3. **[高] 争议表没有任何频控/去重，且写入口不限对象存在性**
   proposal:16-19 无 UNIQUE；proposal:23「任何用户都可以对任意插件提出归属争议」。
   后果：本仓已有成熟限流设施 src/server/platform/rate-limit.ts:46-134（9 个动作，role_application 每天 3 次：:127-133），但新动作不在枚举里 = 无限流。配合无 UNIQUE，一个脚本可对全站 plugins 批量刷争议，撑爆表并淹没真实争议；admin 首页计数 src/app/admin/page.tsx:115 也只统计 plugin_ownerships，争议队列连"有多少待办"都看不到。
   建议：新增 RateLimitAction（如 ownership_dispute：每用户每天 3 次，对齐 role_application），加 UNIQUE(plugin_id, claimant_id)；同一插件同一时间只允许一条 open 争议。

4. **[高] "直接改归属"完全不留审计，与 0042 已加固的审计体系脱节**
   proposal:26 没有任何写 operation_records 的描述。
   后果：operation_records 是全站唯一审计载体，migrations/0042_audit_immutable.sql:17-27 刚刚加了不可删不可改触发器，正是为了让"谁在什么时候做了什么"可信。归属转让是本系统里权限影响最大的一类操作（见第 2 条），不写审计等于给该触发器开了个例外通道；事后无法回答"谁把谁的插件转给了谁"。
   建议：在事务内调 recordOperationInTx（接口见 src/server/modules/operations/service.ts:21-34，含 targetSnapshot/changes/reason），actionType 形如 ownership_transfer；reference 现有用法如 packs/index.ts:937。

5. **[高] status 无 CHECK，状态机无法落地；争议结论不通知当事人**
   proposal:17 `status TEXT NOT NULL DEFAULT 'pending'`。
   后果：本仓所有状态列都有枚举 CHECK（对照 plugin_ownerships 0040:87-88、role_applications 0040:52-53），此处缺失意味着任意字符串可写入，"处理流程为什么没结束"这类问题排查不了。同时 proposal:29「争议有据可查」只承诺可查，没承诺通知；被撤销归属的人可能毫不知情就失去开发者资格（通知机制已存在：notifications 0041:39 的 kind 枚举 + deliverClaimResult ownership.ts:173-200）。
   建议：status 加 CHECK IN ('pending','in_review','accepted','rejected','withdrawn')；结论必须发通知，kind 需在 0041:39 的 CHECK 里扩项（注意 0041 注释 :25-26 已警告"合并顺序是硬约束"）。

## 完整清单（其余）

6. **[中] evidence 列无长度/格式约束，是新的存储滥用面** — proposal:16 `evidence TEXT`。对照 claimed_note 0040:89 有 `length <= 500`。后果：单条争议可写近乎无限文本（SQLite 默认无限制），叠加第 3 条无频控即为放大攻击。建议：limit 500-2000 + 服务层 trim，必要时 `length(evidence) <= N` 的 CHECK。

7. **[中] claimant_id 语义含混，可能与既有 UNIQUE(plugin_id,user_id) 冲突** — proposal:15。后果：0040:111 的 UNIQUE(plugin_id, user_id) 意味着同一用户对同一插件只能有一行 plugin_ownerships，若"争议人"就等于"想认领的人"，那 disputes 与 plugin_ownerships 出现两份平行事实，二者可能不一致（disputes 说 accepted，ownerships 没变）。建议：明确 disputes 只记录"裁决请求"，归属的唯一事实来源永远是 plugin_ownerships；accepted 后必须同事务同步。

8. **[中] 自证循环：争议只能由"想成为归属人的人"发起，缺少第三方/原持有人申诉通道** — proposal:22-23。后果：方案把"举证责任"全压在申请方，但被错误剥夺的一方没有对应入口；proposal:29「归属可纠正」单向。建议：disputes 增加 respondent 视角的状态（被通知、可提交反证），或至少记录当前 verified 持有人的应诉结果。

9. **[中] 迁移缺索引，队列页会全表扫** — proposal:11-20 无 CREATE INDEX。对照 plugin_ownerships 0040:113-115 建了 3 条。建议：按 (status, created_at, id) 建队列索引、按 plugin_id 建查找索引（此条亦可由性能视角给出，不重复展开）。

10. **[低] 缺 created_at 之外的处置痕迹与"谁处理的"** — proposal:16-18 只有 created_at，没有 reviewer_id / reviewed_at / resolution_note。后果：与 0040:90-91 的 reviewer_id/reviewed_at 对照，争议结论无法定位到具体管理员。建议：补齐 reviewer_id、reviewed_at、resolution_note，并在库层加"非 pending 必须有 reviewed_at"的蕴含式 CHECK（写法可照抄 0040:95 与 :110 的注释，那里踩过等价式写法的坑）。

11. **[低] 表缺 ON DELETE / ON UPDATE 行为声明** — proposal:14-15 两个外键都没写。后果：本仓统一风格是显式 ON UPDATE RESTRICT ON DELETE ...（0040:84-85、0039:68-83）。建议：plugin_id 用 CASCADE（与 0040:84 一致），claimant_id 用 RESTRICT/CASCADE 需明确——若沿用 users 相关表惯例多为 RESTRICT。

12. **[低] 方案把"已经存在的能力"说成需要新建的一部分** — proposal:3-4 的现状描述正确，但 proposal:26 的"直接改归属"没有引用既有 reviewClaim（ownership.ts:313-367）与 claimPlugin 的 rejected 复用行逻辑（ownership.ts:259-274）。建议：先从既有服务方法出发描述流程，diff 越小越安全。

## 一句结论
本方案的方向（把争议沉淀成可查记录）是对的，但**三处硬缺口**让它无法安全落地：把归属改写交给"直接改表"（撞唯一索引 + 绕开 taken 检查 + 不审计）、忽略归属与开发者身份的耦合、新写入口既无限流也无去重。建议按第 1/2/3/4 条重写后再审。