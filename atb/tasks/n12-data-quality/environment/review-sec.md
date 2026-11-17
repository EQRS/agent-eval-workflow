# 安全评审报告 — N12《数据校验规则集中化》

评审对象：`tasks/n12-data-quality/environment/proposal.md`（30 行）
评审身份：安全工程师（重点：校验绕过 / 错误提示泄露 / 注入面）
证据基准：本仓 `src/`、`migrations/` 实际实现

---

## 摘要（前 5 条最重要）

1. **【P0】`validatePlugin` 是"假校验"，不是"集中"，是**放宽**——依据 `proposal.md:10-16` 对照 `migrations/0001_baseline.sql:87-96`（`plugins` 表只有 id / repository_id / plugin_path / created_at / updated_at，**没有 name / description / status 列**）与 `migrations/0005_plugin_directory_info.sql:44-51`（display_name / description / author_login / manifest_version / info_updated_at **全部可空、无长度约束**），真实字段名是 `display_name`，不是 `name`；且 `return input as Plugin`（`proposal.md:14`）只做类型断言不剥字段——**任何经过该函数的写路径都会把调用方塞进 input 的任意字段原样放行**，而现状恰恰在 `src/server/modules/catalog/service.ts:513-534` 走的是显式列白名单 INSERT。**建议**：删掉 `as Plugin`，改为"只解构白名单字段重新构造对象"（如 `{ name: String(input.name), description: String(input.description), status }`），并逐入口分写规则。

2. **【P0】唯一一个"能改已收录插件展示字段"的用户级路径会被这条校验**直接**封死或静默污染**——依据 `src/server/modules/catalog/service.ts:496-512`（已收录插件每次提交都会"刷新" display_name / description，除非有 override）、`src/server/modules/corrections/index.ts:25`（纠错白名单只有 display_name / description，允许 200 / 2000 字，**不在其中出现任何 status**）。proposal 把 `status ∈ {active, archived}` 设为所有入口的强制字段且**无默认值**（`proposal.md:13` 用 `!` + includes 判定，undefined 直接 400）：(a) 用户重复提交一个已存在插件最频繁的动作从此一律 400——比现状更严，属功能性回归；(b) 若实现方为了让幂等重放通过而补上 `status: 'active'` 默认值，则**任何用户重复提交都会把管理员归档的插件原地改回 active**（现状 `service.ts:513-534` 的 INSERT 根本不写 status）。**建议**：状态机只允许 `assertCanAdmin` 的后台入口写，用户入口用"状态不变"语义，并显式声明"更新不覆盖管理字段"。

3. **【P0】三处错误提示泄露真实内部资产**——`proposal.md:11-13` 的「名称非法 / 描述非法 / 状态非法」虽是极简文案，但同一个"数据校验集中化"落地时既有的**详细泄露链**仍在，且本方案**没有把 `src/packages/common/errors.ts:44`（"不得包含堆栈、SQL、凭据或第三方原始响应"）写进校验层的契约**：(a) `src/server/modules/imports/index.ts:473`、`:494` 把 `e.message` 原样写进批次 issue 并在 `getBatch`（`:365-377`）回给前端；(b) `src/server/adapters/catalog/github-probe.ts:249-264` 把上游 GitHub 的 `message` 与 `retry-after` 透传成 503 响应体；(c) `src/server/modules/catalog/parse-url.ts:130` 直接把 `/${segments.join('/')}` 非规范化回显。校验集中化恰恰是**统一错误呈现**的最佳落点，方案却只字未提。**建议**：校验层统一抛固定文案 + 稳定 `code` + 字段路径，详细原因只进服务端日志；外部依赖原始报文一律不外溢。

4. **【P1】强制所有入口调用同一个同步校验函数，等于把"逐入口鉴权/频控顺序"从各自 route 里抽走，构成鉴权绕过面**——依据 `src/app/api/admin/imports/route.ts:22`（"先认人再读 body：匿名 + 非法 body 应当是 401/403，不是 400"）与 `src/server/modules/catalog/service.ts:608-611`（服务层二次校验 ctx）。校验集中化后若实现方在 handler 顶部统一先调 `validatePlugin`，未登录用户用非法 payload 就能拿到 400 而非 401，**从而在不登录的前提下探测校验规则与字段形态**，这条正好可以给第 3 点搭成完整的 oracle 链路。**建议**：在 `docs` 与代码注释里把"校验调用点必须在 `assertCanWrite`/`assertCanAdmin` 之后"写成硬约束，并补一条回归用例（匿名 + 非法 body → 401）。

5. **【P1】"删除旧校验"会越过应用层防线，删到数据库约束与触发器层；而这两层恰恰是本库唯一的兜底**——依据 `proposal.md:26`（§3 一键删除）与 `migrations/*` 现状：`plugins` 表**没有任何**字段级约束可删，真正的约束在别处，如 `migrations/0001_baseline.sql:34-36`（content_entities 的 withdrawn/hidden 成对 CHECK）、`migrations/0015_plugin_corrections.sql:54`（`length(value) BETWEEN 1 AND 2000`）、`migrations/0002_triggers.sql:16-30/44-83/90-118`（回指、归属、盖楼）与 `:237-242`（`trg_content_entities_visibility_version`——改 hidden_at 不递增 visibility_control_version 就 ABORT）。**只要有人把"删重复校验"理解成"删 SQL CHECK/触发器"，生效时间是后续所有写路径且不可回滚**。**建议**：§3 明确改写为"仅删除 `src/` 下的**重复 JS 校验**，`migrations/` 的 CHECK/触发器一律视为不可协商的第二道防线"，并加一条静态门禁（`scripts/` 里已有 `verify-migrations.mjs` 同款做法）断言触发器数量与关键 CHECK 仍在。

---

## 完整问题清单

### A. 前提核实（方案所述对象在本仓是否真实存在）

| 方案表述 | 本仓实际 | 结论 |
|---|---|---|
| `proposal.md:10-15` 的 `Plugin` 有 name/description/status | `migrations/0001_baseline.sql:87-96` 只有 id/repository_id/plugin_path/created_at/updated_at；`0005`:44-51 补 display_name/description/author_login/manifest_version/info_updated_at，全部 NULLable | **前提不成立** |
| `proposal.md:21` "用户提交接口" 会写插件行 | `src/app/api/plugins/route.ts:80` → `catalog/service.ts:602` → `runAccept` | 存在，但不是用户"写插件资料"，是提交**收录**（探测 GitHub 后落库） |
| `proposal.md:22` "管理员后台接口" | 没有"创建/编辑插件"的后台接口；只有 `src/app/api/admin/plugins/[pluginId]/overrides/[fieldKey]/route.ts:17-30` 的 DELETE（删 override） | **不存在**可写插件资料的 admin 接口 |
| `proposal.md:23` "批量导入" | `src/app/api/admin/imports/route.ts:17` + `imports/index.ts:248`，最终仍汇入 `acceptPluginSubmission`（`imports/index.ts:481`） | 存在，但与用户入口**共用同一条**路径 |
| `proposal.md:24` "数据修复脚本" | `scripts/seed-demo.mjs:27`、`scripts/acceptance-http-write.mjs:79`、`scripts/acceptance-upload.mjs:79` 直接裸 INSERT `plugins`，**完全不经过任何业务校验** | 存在，且天然已是"绕过校验"形态 |

**核心判断**：本仓真正的"写插件"只有一条业务路径（`runAccept`），外加若干直接 SQL 的脚本。所以 §2 的"四处统一调用"里有两处（用户入口/批量导入）**本来就是同一个函数**，集中化收益近乎为零；而真正的收益点（脚本与业务层校验口径不一致）恰恰被 §2 漏掉了。

### B. 校验绕过

- **B-1【P0】`as Plugin` 断言缺口**（`proposal.md:14`）。返回的是入参对象本身，调用方后续 `Object.keys` / 展开写入即可带入任意字段。对照现状 `catalog/service.ts:513-534` 的 INSERT 是逐列显式白名单，改成本方案后是**负提升**。
- **B-2【P0】状态字段没有归属概念**（`proposal.md:13`）。见摘要第 2 条。同类先例在 `src/server/modules/forum/boards.ts:430`（"is_official 的看板不允许 archived"——管理员改状态有额外闸门），插件如果要支持 archived，必须同样有"谁可以改"的判定，而方案里完全没有。
- **B-3【P1】管理员的 override 通道绕开一切校验**（`corrections/index.ts:56-68` 限 200/2000，但 `plugin_field_overrides.value` 只有 `length BETWEEN 1 AND 2000`，`migrations/0015`:54）。display_name 走 override 可到 2000 字（同步值上限 200，`github-probe.ts:429`），这条路径被方案完全遗漏——"规则一处维护"在最需要的地方漏了。
- **B-4【P1】重复提交即"资料刷新"的隐式写权限**（`catalog/service.ts:496-512`）。任何登录用户重复提交任一已收录仓库的 URL，就会用**当前** GitHub 清单覆盖该插件的 display_name/description（只要没有 override）。这是本仓现存的**用户级隐式写**，方案新增的校验若不加区分，会把这条隐式写"洗白"成合法路径。
- **B-5【P2】脚本层与业务层口径永久分叉**（`scripts/*.mjs:79ff`）。脚本裸 INSERT 不写 display_name 等列 → 页面回退到 `r.full_name`（`catalog/service.ts:873-875`），这正是"数据质量"题的现实病灶；方案把脚本列为"统一调用点"是对的，但**只加校验不加写字段**解决不了它。

### C. 错误提示泄露

- **C-1【P0】校验层没有错误呈现契约**。`proposal.md:11-13` 三个文案本身不算严重泄露，但缺少：(a) 与 `src/packages/common/errors.ts:44` 一致的"不得含内部细节"约束；(b) 与 `src/server/platform/errors-http.ts:55-60` 白名单构造的对接（该函数只放行 code/message/fields/context，**但 message 来自 DomainError，不脱敏**）。
- **C-2【P0】异步批次把底层异常原文回给前端**：`imports/index.ts:473`（`e.message` → issues）、`:494`（同）、`:365-377`（`issuesOf` 原样输出给 `GET` 批次接口）。校验集中化后如果校验函数混进 `imports` 的 try 里，任何非 DomainError 的提示（含 SQLite 报错）都会进 issues。
- **C-3【P1】上游报文透传**：`github-probe.ts:249-264` 把 GitHub 的 `message` 与 `retry-after` 拼进 503 响应。这是给攻击者的"我方配额/令牌状态"探针；同时也是"方案外但属于本次校验落点周边"的真实泄露点。
- **C-4【P2】非规范化回显**：`parse-url.ts:104-108`（owner/repo 直接回显）、`:130`（整个 path 拼接回显）。有 `OWNER_RE`/`REPO_RE` 的前置正则，但回显发生在**校验失败那一支**，实际是把用户输入原样反射——建议截断 + 固定模板。

### D. 注入面

- **D-1【核实无问题】SQL 注入：全仓无 `.exec()` 拼接可注入面**。`catalog/service.ts:227-229` 的 `escapeLikePattern` 处理 LIKE 通配，`:97-99`（`overridesFor`）、`corrections/index.ts:97-100` 的 `IN (${placeholders})` 只拼 `?` 占位符数量，绑定走参数化——**安全**。
- **D-2【P1】序号与列名拼接面**：`catalog/service.ts:501-512` 用 `setClauses.join(', ')` 拼 UPDATE 列名。当前列名是代码常量（安全），但**校验集中化后如果引入"根据 input keys 动态拼列"的通用写法，这里就是唯一能变成注入面的位置**。建议在方案里明确"UPDATE 的列名必须来自编译期常量数组，禁止来自输入键"。
- **D-3【P2】存储型注入（HTML）**：`src/app/plugins/[pluginId]/page.tsx:117/155` 与 `plugins/page.tsx:134` 用 JSX 渲染 `displayName`/`description`，React 默认为文本节点，**无 XSS**；但该页无 `generateMetadata`（`grep metadata` 零命中），一旦后续补 metadata 且用未校验值进 `<title>`/`og:title`，2000 字未过滤描述会成为诱饵面。校验层的长度上限（name≤200 / description≤2000）应与 `github-probe.ts:112-117` / `:429-430` 的截断上限一致，避免"通过校验但上游已截断"的口径分裂。

### E. 与数据库第二道防线的关系（关系到"注入/绕过"的兜底）

- **E-1【P1】** `plugins` 表**本身没有字段级约束**：`0001_baseline.sql:87-96`（无长度/非空 CHECK）、`0005`:44-51（全 NULLable）。这意味着**应用层是 plugins 展示字段的唯一防线**——正好是"集中化"的必要性来源，但反过来也意味着**这个函数一旦写错，没有任何一层兜底**。方案应该明确承认这一点，并配套在 `migrations/` 补最低限度的 CHECK（如 `display_name IS NULL OR length(display_name) <= 200`）。这是本方案**最该做而完全没做**的一环。
- **E-2【P1】** `migrations/0002_triggers.sql` 现有 11 个触发器承担跨行归属校验（`:16-30` 回指、`:44-83` 修订指针、`:90-118` 盖楼、`:237-242` 可见性版本自增）。§3"删除重复校验"若被执行到 SQL 层，违规数据可以"看起来存在、实际孤儿"。参照 `:130-133`（content_entities 的 pack_release 切片闸门），本库的立场历来是"宁可多一层也不放行"。
- **E-3【P2】** 迁移里可借鉴的强约束写法：`0001_baseline.sql:94`（`UNIQUE (repository_id, plugin_path COLLATE BINARY)`）、`0005`:27-28（`json_valid(topics)`）。建议把 name/description 的长度约束照此补进迁移，作为 O-1 的具体落点。

### F. 调用点顺序（鉴权 → 频控 → 校验 → 探测）

- **F-1【P1】** 现状顺序有明确规范：`plugins/route.ts:63-73`（频控**必须**在出网探测 GitHub 之前）、`admin/imports/route.ts:22`（先认人再读 body）、`admin/plugins/.../overrides/route.ts:24`（先 `assertCanAdmin`）。集中化后**必须**把这个顺序写进方案，否则"校验"很容易被放到 handler 第一个 `await` 之前，变成未鉴权可达的规则探测面（配合 C-1 形成完整 oracle）。

---

## 建议的修订方案（最小可行）

1. **§1 重写**：删除 `as Plugin`；函数返回**新构造的白名单对象**，不返回入参；长度上限来自与 `github-probe.ts:112-117` 同源的常量；错误输出 = `{ code:'input_invalid', message:'输入无效', fields:[{path,message}] }`，详细原因 `log.warn`。
2. **§2 分表**：按"入口 × 可写字段 × 是否可改状态"列一张矩阵。用户入口：只读同步字段、状态不可写；管理员入口：状态可写且有额外闸门（参照 `forum/boards.ts:430`）；脚本：走"写库时跳过业务校验 + 事后 `verify-*` 断言"（仓里已有 `scripts/verify-migrations.mjs`、`verify-tests-bite.mjs` 的先例）。
3. **§3 收口**：只删 `src/` 的重复 JS 校验；`migrations/` 的 CHECK/触发器列为不可删；并补 `plugins.display_name/description` 的长度 CHECK（E-1）。
4. **新增 §4 错误契约**：明文禁止把 `e.message`（`imports/index.ts:473/494`）与上游报文（`github-probe.ts:249-264`）透传给客户端的批次接口与响应体。
5. **新增 §5 回归用例**：匿名 + 非法 body → 401（不是 400）；重复提交不改变 admin 归档态；override 值不得绕过长度校验；触发器数量守卫。

---

## 结论

方案方向（集中化）正确，但**落地对象错了**：它按"存在一张带 name/description/status 的 plugins 表、有四个不同强度的写入口"来设计，而本仓只有一条业务写路径 + 全 NULLable 无约束的列 + 若干裸 SQL 脚本。按现方案执行，最好的结果是无效重构，最坏的结果是**把校验最弱的一环（`as` 断言 + 无归属的状态字段"）变成全站唯一关口，同时删掉实际承担兜底的触发器与 CHECK**。建议打回重写 §1/§2/§3 后再评审。
