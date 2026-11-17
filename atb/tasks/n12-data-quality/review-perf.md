# N-12 性能评审（性能工程师）：数据校验规则集中化方案

## 摘要（前 5 条最重要）

1. 【严重】「一处校验」在本仓已经是既成事实，方案改的是**最便宜的那一层**，而它要动的调用点恰恰是**上游配额最紧的那条链路** — 依据 proposal.md:8-15（一次性全字段校验）、proposal.md:19-23（调用点含三处既有管线）、src/server/adapters/catalog/github-probe.ts:337-382（findPluginDirs 每次读树后最多 12 次 fetchPluginManifest）、src/server/adapters/catalog/github-discovery.ts:178-220,346-399（findPlugins 每仓库 1 次 tree + 每 package.json 1 次，上限 80，每次失败退避重试 ×3）、src/server/modules/catalog/service.ts:359-418（收录检查再走一遍网络：仓库 + 清单 + patch）、src/server/modules/catalog/discovery.ts:55-81（未认证 maxRequests=58/时，认证 400/轮）、src/server/modules/imports/index.ts:479-486（apply 复用同一路径）— 建议：规则模块做成**纯同步、无副作用、可短路的纯函数**，先本地预校验（长度/枚举/形状/requestKey 黑白名单）**在出网之前**直接拦截，N 次上游请求降为 0；并加白名单：findPluginDirs 路径不跑完整字段校验（它只需要「是否声明 dsh.bundle」，长描述本就不影响结论）。

2. 【严重】方案说「代码重复」，但仓库里**根本不存在这份校验**，集中后的新规则会变成**新行为**，直接卡死存量脏数据 — 依据 proposal.md:4（「各处都在做手动校验，代码重复」）、src/server/modules/catalog/service.ts:480-535（收录/刷新插件**全路径无一处 name/description 长度/枚举校验**）、src/server/adapters/catalog/github-probe.ts:112-117,321-334,428-433（描述按 2000 截断）、src/packages/common/field-rules.ts:27-30；且 `status ∈ {active, archived}` 在 schema 里**不存在**：migrations/0001_baseline.sql:87-95 的 plugins 只有 id/repository_id/plugin_path/时间戳，0005_plugin_directory_info.sql:44-51 只有可空的 display_name/description — 建议：先 `SELECT` 全量盘点存量越界行数（这是评估成本的前提，不是事后补），再定「新数据拒绝、旧数据治理」两种策略；status 一项要么从方案删掉，要么先加列 + 迁移 + 默认值。

3. 【高】最贵的重复校验不在插件 CRUD，而在「发现/导入/回填」三条路径对同一批数据**重复走完整收录管线（含网络）**，逐条各开一个事务、逐条重放 — 依据 proposal.md:8-26 完全没提这三条路径、src/server/modules/imports/index.ts:433-438 vs 481-485（先一次 `findExistingPlugin`，没命中就进另一次 `acceptPluginSubmission`，后者再查一遍同一个幂等键）、src/server/modules/imports/index.ts:230-244（每条结果一个独立 `withTransaction`）、scripts/backfill-plugin-info.mjs:8-20,99-103（对 `info_updated_at IS NULL` 的每条存量重跑一次真实收录检查，每条 3+ 次 GitHub 请求，无批次预算）、src/server/modules/catalog/discovery.ts:148-152（`input.maxPlugins ?? DISCOVERY_DEFAULTS.maxPlugins` 用未分档常量兜底）、src/server/adapters/catalog/github-probe.ts:346-356（findPluginDirs 在 service.ts:379 之后又读一遍 /repos/{o}/{r} 与整棵 git tree）— 建议：把「已存在短路」提到**出网之前**并合并成一个查询（`repositories.github_repository_id ↔ plugins.plugin_path`），删掉 imports 里的预查或让它成为唯一查询；把校验结果挂在幂等键上做二次确认缓存；回填脚本加 `--limit` 之外的**累计请求预算**。

4. 【高】"不要动生产数据却动了"：'--apply 安全'靠的是校验不存在这一巧合，集中校验会把它变成一个**半途终止且没有进度报告**的批处理 — 依据 proposal.md:22（数据修复脚本）、scripts/backfill-plugin-info.mjs:44-47,99-119（接受 `userId: null`，借用**库里第一个管理员**的身份写操作记录）、src/server/modules/catalog/service.ts:530-564（每次刷新都会 `UPDATE` 插件行：`updated_at/info_updated_at = now`，并写一条 `operation_records` 审计）— 建议：修复脚本改用**专用 actor**（system + taskId，见 service.ts:666-696 的 `acceptPluginSubmissionBySystem`），不要借用管理员身份；校验失败必须**累计并继续**、最后一次汇总，不许 fail-fast。

5. 【中】规则一旦进热路径就在**每次请求 × 每次页面渲染**上付钱，而方案没有幂等性/分阶段约束 — 依据 src/packages/common/field-rules.ts:16-30（现有纯判定集中在 30 行内，无 I/O、无正则多轮回溯）、src/server/platform/tx.ts:37-52（写事务内网络调用与校验混在一起会拉长持有时间）、migrations/0014_review_search.sql:18-24（`trigram` 只对 ≥3 字符查询生效，校验不应与检索口径互相借用）— 建议：校验分三层（Wire→Domain 归一 / Domain 不变量 / Persistence）并在架构上**幂等**，入口调用两次不产生第二次 I/O；长度上限来自常量而不是裸数字（comments/index.ts:60-110、forum/posts.ts:349-360 都是常量先行）。

---

## 逐项明细

### P-1【严重】性能成本归因错位：真正的放大在"上游请求次数"，方案动的却是最便宜的本地分支

- **依据**：
  - 方案（proposal.md:8-15）的校验体是 O(1) 字符串比较与长度判断，执行成本是纳秒级；它在整条收录链里的占比可以忽略。
  - 收录链的真实成本在出网：github-probe.ts:268-334 一次 `fetchRepository`，:384-468 一次 `fetchPluginManifest`（其中再带一次 patch 存在性 GET，:448-450），service.ts:361-418 串行执行；discovery.ts:55-81 把未认证上游预算卡在 **58 次/小时**（认证 400 次/轮），github-discovery.ts:30-35 默认重试 2 次、退避 400ms×2^n。
  - 更糟的是 findPlugins（github-discovery.ts:355-361、364-366）对**每个** package.json 都要一次 `contents` 请求，上限 80 —— 这些路径**根本不需要** "name ≤ 100 / description ≤ 5000 / status 枚举" 任何一项。
- **后果**：把统一校验塞进「批量导入」（proposal.md:22）之后，用户真正感知的变化是：一条不合格的条目**仍然**会把 3 次 GitHub 请求烧完才被拒（imports/index.ts:481-486 里校验发生在 `acceptPluginSubmission` 内部），在 58 次/小时的匿名档下等于**第一轮只能处理不到 20 条**，其余全部 "预算已用尽留待下一轮"（imports/index.ts:468-471）。方案对外宣称的是"代码更简洁"（proposal.md:28-30），实际交付的是吞吐下降。
- **建议**：
  1. 校验模块必须是**纯函数且无 I/O**，绝不允许在服务内部触发 fetch；
  2. 新增一道 **pre-flight 阀门**：在 `runAccept` 与 `runBatch` 调用探针**之前**跑"零成本可判"的规则（长度上界、枚举、requestKey 非空白、url 形状），不通过就直接 `err.input`，不消耗预算；
  3. findPluginDirs 这条辅助路径（probe.ts:337-382）**显式豁免**完整校验，只判"是否声明 dsh.bundle"，避免用 12 次请求去否决一个不影响结论的字段。

### P-2【严重】"删除重复校验"在性能上是负收益：现在没有重复可删，删的是**真实的防线**

- **依据**：`grep "export function (validate|assert|normalize)"` 在 `src/` 只命中 10 个函数，全部是**领域专用**的（`assertSafeStorageKey` file-store.ts:42、`normalizeCategorySlug` catalog/service.ts:72、`normalizeCategoryTopics` :93-116、`assertCanWrite` context.ts:34 …），没有一处是"插件字段校验"的重复副本；plugins 的写入路径 service.ts:480-535 从头到尾没有字段级校验。
- **后果**：方案第 3 节（proposal.md:25-26「删除各处的重复校验代码」）在插件链路上**没有可删的代码**；实操中唯一可能被当成"重复"删掉的，是 `field-rules.ts` 里的 `isNonBlank`/`isUuidV7` 这类**被多处调用的真实约束**，以及各个模块各自带的业务上界（comments/index.ts:105-110 的 `COMMENT_BODY_MAX_LENGTH`、forum/posts.ts:349-360、feedback/index.ts:187-199）。删掉它们 = 每次写请求多一次"越界数据入库 → 再被 DB CHECK 拒绝 → 返回 500 而不是 400"的往返。
- **建议**：把"集中"定位为**新增约束 + 复用判定原语**，不是"删除现有代码"。给每个模块的既有上界一个迁移路径（常量 → 从统一模块 re-export），而不是删函数。

### P-3【高】发现 / 导入 / 回填三条管线对同一份数据的重复计算，比方案里任何一条规则都更值得先做

- **依据**：
  - **同一次收录里查两遍**：imports/index.ts:433-438 先 `findExistingPlugin`（一次 JOIN），没命中才进 `:481` 的 `acceptPluginSubmission`，后者在 service.ts:487-489 又查一次同一个 `(repository_id, plugin_path)`。
  - **同一次收录里读两遍仓库**：service.ts:361 已取 `repoInfo`，`:379` 走 findPluginDirs 时 probe.ts:348 又取一次 `/repos/{o}/{r}`（含 `default_branch`），再 `:353` 取整棵 tree —— 而 scan 结果紧接着被 service 的 `:371-375` fetchPluginManifest 重新读一遍相同的 package.json。
  - **批处理 N 个事务**：imports/index.ts:230-244 每条结果单独 `withTransaction`（better-sqlite3 同步驱动，db.ts:6-8），`targets` 最多 `budget.maxPlugins` 条，每条至少一次写事务 + 一次读；`:499` 又 `readItems` 全表读一遍只为数 pending。
  - **批量回填无预算**：backfill-plugin-info.mjs:64-72 取全量 `info_updated_at IS NULL`，循环 `:87-112` 串行、每条真出网（`:99-103`），没有 `discoveryBudgetFor` 这类闸门，也不检查 `wasRateLimited`。（imports 侧是有预算的：`:400,416-417`。）
- **后果**：在匿名档下，一层"查两遍"就吃掉 58 里的 1-2 次；回填脚本一次跑完全量会在一小时内把 60 次配额打穿并静默 skip 过半。若方案第 2 节真的把统一校验接到这四处（proposal.md:19-23），这些成本会被**固化成"标准做法"**。
- **建议**：
  1. 把"已存在判定"收敛为一个函数，在**出网之前、用一次查询**完成，并作为 `runAccept` 的参数传入（避免二次查库）；
  2. 让 findPluginDirs 复用调用方已取得的 repo 元数据（把它改成显式接收 `defaultBranch`，而不是自己再 GET 一次）；
  3. 回填脚本接 `discoveryBudgetFor(process.env)` 与 `wasRateLimited()`，与导入/发现共用同一份上游预算，避免三条路径各打满一次小时配额。

### P-4【高】"数据修复脚本也走统一校验"会让一次修复在中途静默停下，且审计归属错误

- **依据**：proposal.md:22 明确把数据修复脚本列为调用点；backfill-plugin-info.mjs:99-103 用的是一个**真实管理员**的身份（`:49-56` 选角色最高的活跃用户），service.ts:530-564 在每次刷新时 `UPDATE plugins SET updated_at=?, info_updated_at=?` 并写一条 `operation_records`。
- **后果**：集中校验一旦生效，存量里最需要修的恰恰是最容易被校验拒掉的（超长描述现在允许 2000，规则却按 5000 甚至更严），脚本会 `skipped += 1` 继续（`:106-111`），但**没有任何"这是最后一次运行"的汇总出口**——人工只会看到"跳过 30 条"然后重跑，而重跑同样跳过（幂等但无进展）。同时审计里出现的是管理员名字，事后追责链是错的。
- **建议**：修复脚本走 `acceptPluginSubmissionBySystem`（service.ts:666-696，system + taskId 的既有入口），并在校验层提供 `mode: 'repair'`：**只告警不阻断 + 结果汇总按规则维度分组**。这不是"绕过校验"，是把"拒绝"与"报告"两个动作分开——性能上也是必要的，否则一次全量回填要跑两遍（一遍发现不合格、一遍修复）。

### P-5【中】集中校验一旦进热路径，需要考虑的是"重复调用"与"纯函数性"，方案未提

- **依据**：`field-rules.ts` 现在只有 30 行纯判定（:16-30），没有分支外的正则回溯、没有 `JSON.parse`；而方案给的是 `input.name.length > 100` 之后再 `input.description.length > 5000` 这样的**顺序短路**（proposal.md:11-14），方向是对的，但没有约定"只能被调用一次"。
- **风险点**：Next 的 RSC 渲染与 API 路由在同一个动作上会各调一次（仓库里 50+ 页面 `force-dynamic`），若统一校验放在**读路径**（如 service.ts:698-782 `getPlugin` 返回给组件前再校验一遍），就会把 O(1) 乘以每次渲染的调用密度；此外 `withTransaction`（tx.ts:42-52）内的任何非必要工作都会延长 SQLite 写锁持有时间。
- **建议**：规则模块声明三条性质并写进注释与测试：(a) 纯函数（同入同出、无 I/O、无时钟）；(b) 幂等（连续调用两次结果一致、无副作用）；(c) 只放在**写入口**（`acceptPluginSubmission` / `createBatch` / `runBatch`），**不放在读路径**与事务内部。

### P-6【中】长度上限必须来自常量与既有口径，而不是方案里凭空写的 100/5000

- **依据**：probe.ts:112-117 对 description 按 **2000** 截断、name 按 **200**（:428-433）；discovery 适配器同样 2000/200（github-discovery.ts:142-147,393-395）。方案给的是 name ≤ 100、description ≤ 5000（proposal.md:11-12）。
- **后果**：两条口径并存 = 同步写入的值（≤2000）永远合规，而用户在别的地方手填 3000 字会被拒 —— 同一字段两条阈值，属于"集中化"最想消灭的那种重复，只是换了个位置。反过来，若某条路径真的能落 5000 字，`getPlugin` 渲染大描述又会落到 catalog/service.ts:816-826 的列表查询上（那里每行会带 description）。
- **建议**：把上限定义在 `field-rules.ts` 同一处置为唯一常量，并让 probe 的截断值**引用同一常量**（`asTextOrNull(v, FIELD_LIMITS.pluginDescription)`），否则"集中"只是多了一层。

---

## 附：本次评审的核实范围

- 读过：proposal.md 全文（30 行）、instruction.md、criteria.md、catalog/service.ts、catalog/discovery.ts、imports/index.ts、adapters/catalog/github-probe.ts、adapters/catalog/github-discovery.ts、platform/tx.ts、platform/errors.ts、packages/common/field-rules.ts、migrations/0001_baseline.sql、0005_plugin_directory_info.sql、0014_review_search.sql、scripts/backfill-plugin-info.mjs。
- 未做：未运行压测/基准（任务约束为只评审、不改代码）；上述数字（58 次/时、400/轮、2000 截断、80 个 manifest 上限）均直接取自代码常量，未实测。
