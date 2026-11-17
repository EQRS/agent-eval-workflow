# N-46 评审报告 · T-SRE（运维）（Agent Teams 复用）

> 测评：官方 Agent Teams · 成员 b2-e1-a（复用）
> 日期：2026-11-17

## 摘要（前 5 条）

1. 【严重】未定义「改归属」的事务边界与并发语义
   - 依据：proposal.md:26 + 0040:111 UNIQUE(plugin_id, user_id) + 0040:119 部分唯一索引
   - 建议：单事务「撤销旧 verified + 建立新 verified」，顺序不能反，且要处理两处唯一约束

2. 【严重】争议表无审计字段、无状态机、无容量约束
   - 依据：现有 operation_records 有 actor_kind/target_snapshot/changes/reason 一整套
   - 建议：处理结论落 operation_records，别另起炉灶

3. 【严重】status 无 CHECK、created_at 无 >=0、缺索引
   - 依据：对照 0040:52-53、0040:58、0040:64-65

4. 【中】迁移 0043 新表是否纳入备份未知
   - 依据：deploy/dsh-review-backup.service:19 用 backup.mjs --keep 14
   - **T-SRE 主动标注：我未读 backup.mjs，无法确定是整库拷贝还是按表列举**

5. 【中】「任何用户可提争议」缺频控与幂等
   - 依据：rate_limiter_flexible（0006）+ rate-limit.ts

## 其余 5 条

6. 直接 UPDATE 会绕过 0040:96-110 的踩坑保护
7. **无回滚预案**：项目无 down 迁移，备份每日 03:00 一次 → 上线当天出事会丢最多 33 小时数据
8. evidence TEXT 无长度上限
9. 多实例并发（当前单实例）
10. 无监控：争议积压量应进 health ?detail=1

## 整体判断

> 方案覆盖了「做什么」，但**完全没有覆盖「怎么做才不出事」**。