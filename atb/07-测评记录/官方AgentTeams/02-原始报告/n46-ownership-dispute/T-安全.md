# N-46 评审报告 · T-安全工程师（Agent Teams 复用）

> 测评：官方 Agent Teams · 成员 b-e1-a（复用）
> 日期：2026-11-17

## 摘要（前 5 条）

1. 【严重】处理环节没有鉴权要求，「管理员」如何判定未定义
   - 依据：proposal.md:26；对照 reviewClaim 用 isAdmin 守卫（ownership.ts:314）
   - 后果：若实现时漏掉，**任何登录用户都能直接改归属**——提权漏洞
   - 建议：路由层 assertCanAdmin + 服务层 isAdmin 双保险

2. 【严重】「任何用户可对任意插件提争议」= 无频控、无去重、无归属前提
   - 依据：proposal.md:23；对照 plugin_pack_appeals（0028:32-46）+ reports 防刷索引（0012:59-62）
   - 后果：低成本 DoS，几百个请求即可淹没真实争议
   - 建议：UNIQUE(plugin_id, claimant_id) WHERE status='pending' + 日限额 + 限定申请人资格

3. 【严重】直接改归属绕过 claimPlugin 的 taken 冲突检查
   - 依据：ownership.ts:245-250（verified 被占即 err.conflict）；0040:86（source 区分 github_match/claimed）
   - 后果：同时破坏「一插件一 verified」与来源真实性
   - 建议：走独立 disputeResolve 方法，复用同一套事务与冲突检查

4. 【高】状态机缺约束（可利用的状态注入）
   - 依据：proposal.md:17 status TEXT 无 CHECK；对照 0040:87-110 有严格枚举 + 蕴含式 CHECK
   - 建议：CHECK (status IN (...)) + resolved_by/resolved_at/resolution_note + 待处理态不得带处理痕迹

5. 【高】evidence TEXT 无长度上限 + 审计链缺失
   - 依据：0040:89=500、0012:50=2000、0028:32=5-2000；方案无 operation_records
   - 建议：evidence 限长；处理后写 operation_records；表内保留处理痕迹

## 其余 3 条

6. 【中】FK 行为未指定（表默认 RESTRICT vs ownerships 用 CASCADE）
7. 【中】管理队列缺索引（对照 idx_reports_queue 0012:58）
8. 【低】未定义原持有人处置与通知 → **归属被悄悄转移，攻击窗口可无限延长**

## 结论

> 方案目标合理，但**当前形态不满足该项目的安全基线**。