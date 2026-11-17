# 「开源许可证合规扫描」方案评审

## 评审基线
- 方案所指依赖清单确实存在：[package.json](<仓根>/package.json) 第 25–50 行（dependencies 12 项 + devDependencies 10 项）。
- 仓库**没有 LICENSE 文件、没有 NOTICE、没有 COPYING**（全仓 glob，node_modules 外零命中）——本项目自身发布许可证未定，"合规"缺少基准。
- 全仓 `src/` 只有一处提到许可证文案：`src/app/spec/page.tsx:54` 的 `'MIT / BSD / Apache'` → `'开源许可'`，那是站内**用词替换表**，不是许可证白名单。
- 数据库没有任何许可证字段：`migrations/0001_baseline.sql:71-84`（repositories）与 `:87-96`（plugins）只有 ID、仓库 ID、full_name、可用状态、plugin_path。

---

## P0：会让结论错误或根本跑不起来（7 条）

**1. 白名单口径把仓库自用依赖判成不合规（方案 §2，第 16 行）**
① "只允许 MIT / Apache-2.0 / BSD 三种"——这个仓装着 `typescript`、`better-sqlite3` 等强 copyleft，但更直接的问题是：`next: 16.4.0`（[package.json:27](<仓根>/package.json#L27)）在 `devDependencies` 里的 `@types/*`、`tsx`、`vitest`（[package.json:39-50](<仓根>/package.json#L39-L50)）是**开发期依赖**，按方案口径一并卡死，第一天 CI 就是红的。
② 后果：方案上线即失败，团队第一反应会是"放宽白名单"，而不是"看清许可证义务"——白名单一旦被削弱，GPL 就跟着漏进来。且 `license-checker --summary`（第 12 行）只输出每种许可证的**包计数**，它连"哪个包是 GPL"都不告诉你，无法按包分级。
③ 建议：按「分发形态」分三档——参与产物构建的 runtime 依赖（strict）、devDependencies / 构建工具（宽松）、以及**本仓实际是 npm `private: true`**（[package.json:4](<仓根>/package.json#L4)）需先明确"我们到底分发什么、以什么许可证分发"，白名单由这个答案倒推，而不是先定名单。

**2. 误判根因：许可证合规 ≠ 白名单匹配，义务被完全忽略（方案 §2）**
① 方案把合规简化为"许可证字符串 ∈ 集合"。MIT/Apache-2.0/BSD 恰恰都是**有义务**的：MIT 与 BSD 要求随分发保留版权声明与许可证全文，Apache-2.0 还要求带 NOTICE 文件、保留变更声明、附带明确专利授权。
② 后果：方案宣称的 "自动发现不合规依赖"（第 22 行）交付不了——全部包都在白名单内时它报"合规"，但镜像里**可能一条版权声明都没有**。实际风险落在分发物上：见第 3 条。
③ 建议：白名单降级为"许可证类型检查"，另加两条独立检查：每个 runtime 依赖的 LICENSE 文本是否进入分发物（本仓为容器镜像，见第 3 条）、是否有 NOTICE 需要转抄。

**3. 扫描对象错位：真正分发的是 Docker 镜像，不是 package.json（方案 §1、§3）**
① 第 9 行 "用 license-checker 扫描 package.json"——但生产发布物是容器镜像：[docker/web.Dockerfile:57](<仓根>/docker/web.Dockerfile#L57) 把整个 `node_modules` 原样 `COPY` 进 runner 层并随镜像分发；[docker/web.Dockerfile:14](<仓根>/docker/web.Dockerfile#L14) 的基础镜像 `node:24-bookworm-slim` 里还带 Debian 发行包与 Node 自身；[docker/compose.yaml:44-68](<仓根>/docker/compose.yaml#L44-L68) 的 web 服务是常驻对外入口。
② 后果：只扫 npm 树，**完全漏掉** Debian 用户态与 Node 二进制这两块的分发义务（Debian 包通常要求在文档目录附版权声明）；镜像层里的 LICENSE 文件在 `npm ci --omit=dev` 之外是否齐全也无从核对。更滑稽的是方案连"我们自己以什么方式对外分发"都没写，所以根本无从判断谁需要满足什么义务。
③ 建议：把扫描的**判定对象**从 "package.json" 改成 "发布物清单"：npm 生产依赖树 + 基础镜像 OS 包 + 随镜像携带的 `/app/bin` 二进制（[docker/web.Dockerfile:65](<仓根>/docker/web.Dockerfile#L65)，目前只有 README/占位）——这三部分会各自带来不同的许可证义务，且基础镜像那部分需要额外的 OS 包扫描器。

**4. 扫描的是声明值，不是库里的实际许可证文本（方案 §1）**
① `license-checker` 读的是各包 package.json 的 `license` 字段（[package-lock.json:45](<仓根>/package-lock.json#L45) 起可见大量 `"license": "MIT"` 元数据），以及 `node_modules` 里已解出的包。
② 后果：(a) 声明的许可证与仓库里 LICENSE 文件不一致时不会被发现，而这正是供应链改证最常见的形态；(b) 它依赖 `node_modules` 目录现状——本仓 `npm ci` 是带 `--include=dev` 的全量安装（[docker/web.Dockerfile:33](<仓根>/docker/web.Dockerfile#L33)），但 CI 环境里若目录状态不同（skip-install、缓存、optional 平台包缺失），结论会随环境漂移；(c) 平台特有可选依赖（如 `@next/swc-win32-x64-msvc`、`lightningcss-win32-x64-msvc` 这类）在 Linux CI 上根本不下载，**扫不到但会随对应平台的生产分发出去**。
③ 建议：以 `package-lock.json`（含 integrity 哈希，不受本地安装状态影响）作为包清单的真源，逐包比对"声明 license 字段"与"包内 LICENSE 文件实际文本"，两者不一致即告警；可选平台依赖显式纳入分析而非依赖安装结果。

**5. 无 baseline，方案第一次运行必然红、也证明不了"新增"（方案 §3，第 19 行）**
① 第 19 行 "在 CI 里加一步，不合规就失败"。但接入时存量依赖已一大片，第一次 PR 就是红的。
② 后果：红着的门禁三天就会被 --continue-on-error / 注释掉，合规检查归零；这与本仓现状正好相反——[.github/workflows/ci.yml:27-34](<仓根>/.github/workflows/ci.yml#L27-L34) 的三个门禁（typecheck、test、用词）**都是全绿才推进**，且 ci.yml:3-5 的注释明确写着"629 条用例只靠提交前记得跑"的教训，就是为了不让任何人习惯性忽略红色。方案第 19 行不写 baseline，等于给 CI 塞进第一个"永远红"的步骤。
③ 建议：采用 baseline 文件（先快照当前树 → 只对**新增/变更**依赖报错），并把 baseline 的更新做成需要人工 review 的显式动作，而不是自动刷新（自动刷新会把真实的许可证变更一起洗白）。

**6. 没有触发时机：只会扫 main 上的树，扫不到"即将进入"的依赖（方案 §1、§3）**
① 方案只描述"在 CI 里加一步"，没写触发点、没写锁文件。
② 后果：真正的风险窗口是 PR 引入新依赖那一刻，以及上游**静默改证**那一刻（同一个版本号、下一次 `npm ci` 装出来就是新许可证——本仓锁文件只是版本号+哈希，不管许可证）。方案对"上游改证"这一类最典型的供应链形态零覆盖。
③ 建议：三层触发——PR diff 时只扫新增依赖（快、可阻断）；每日定时扫全树并与上次快照 diff（捕获改证）；发布前扫镜像最终层（见第 3 条）。

**7. "不合规就失败"没有处置路径，等于把决策权丢给 CI（方案 §3）**
① 第 19 行只有"失败"两个字。
② 后果：遇到确实必要的 GPL 类依赖（或 `typescript` 自带那套版权声明要求）时，团队唯一能做的就是改白名单或者禁用 CI——两个动作都不留证据，事后无法回答"当初是谁批准放行的、依据是什么"。本仓已有 `operation_records` 表做管理动作留痕（[migrations/0001_baseline.sql:411](<仓根>/migrations/0001_baseline.sql#L411) 起），合规豁免没有同样机制。
③ 建议：定义例外流程——例外必须以显式配置文件（含包名、许可证、批准理由、到期日）落库/入库评审，CI 遇到例外放行并在输出里点名"本次运行利用了哪几条例外"，避免例外静默累积。

---

## P1：工程细节会咬人（6 条）

**8. `npx license-checker` 不可复现（第 12 行）**
① 未固定版本。`npx` 会解析并可能拉取最新版；且本仓依赖是用 `npm ci` 按 package-lock 精确安装的（[.github/workflows/ci.yml:25](<仓根>/.github/workflows/ci.yml#L25)，ci.yml:18-19 的注释甚至专门警告过"混用包管理器会生成第二套锁文件"）。
② 后果：扫描器自己成了不受 locking 管控的供应链入口；且 CI 里 `npx` 运行时若被要求
stall 会与 `NODE_ENV=production`（[docker/web.Dockerfile:15](<仓根>/docker/web.Dockerfile#L15)）相互摩擦。这与本仓已经反复强调的"只用 npm、只用锁文件"纪律直接冲突。
③ 建议：把扫描器作为固定版本的 devDependency 进 package-lock，用 `npm exec --` 调用；或在 CI 里用固定 tag 的独立 action（`license-compliance-action` 等）但同样固定到 commit SHA。

**9. `--summary` 的输出不可用于门禁（第 12 行）**
① `--summary` 是给人看的汇总（"MIT: 500, ISC: 300, GPL-3.0: 2"），包含包名的地方被折叠掉了。
② 后果：报错说"有 2 个 GPL"，但没告诉你是哪 2 个——CI 日志里无法定位、无法加例外、无法写进故障单。本仓的三个门禁用词检查是直接给出文件行号的（[.github/workflows/ci.yml:34](<仓根>/.github/workflows/ci.yml#L34)），新门禁的信息质量会明显低于现有水准。
③ 建议：用 JSON 输出（`--json`），自己写一个薄脚本做白名单判定 + 逐包列出违规项，并在 CI 里做注解（`::error file=...::`）。

**10. 白名单把常见合规许可证误伤为违规（第 16 行）**
① 只列 MIT / Apache-2.0 / BSD。而 `BSD` 是不存在的一串字，实际是 `BSD-2-Clause` / `BSD-3-Clause` / `0BSD`；Apache-2.0 的 SPDX 标识必须带后缀（`Apache-2.0`，不是 `Apache`）；npm 生态里大量包的标识是 `ISC`、`CC0-1.0`、`Unlicense`、`BlueOak-1.0.0`，这些都是**比 MIT 更宽松**的。
② 后果：把更宽松的许可证判为违规，等于制造大量假阳性，直接导致第 5 条说的"门禁被关掉"。字符串匹配还会漏掉 `MIT OR Apache-2.0`（SPDX 表达式，必需选其一时应视为合规）、`SEE LICENSE IN <file>`（必须人工看）这类写法——后者最危险：它**根本不告诉你是什么许可**，而按方案它能"通过"还是"不通过"取决于它恰好不匹配白名单，属于碰运气。
③ 建议：用 SPDX 表达式解析（`spdx-expression-parse`）而非字符串比较；`OR` 表达式只要有任一分支被允许即放行；`SEE LICENSE IN` 与未知标识一律进入人工队列并阻断到有人签收。

**11. 许可"来源"没覆盖：GitHub 来源的 DSH 插件（第 4 行"依赖很多第三方包"）**
① 方案只想到 npm。本仓的业务恰是收录 DSH 插件，安装目标支持 GitHub 提交固定（[src/packages/common/pack-install.ts:20-22](<仓根>/src/packages/common/pack-install.ts#L20-L22)：`kind:'github'` + `commit`），而数据库里 `plugins`/`repositories` 没有许可证列（[migrations/0001_baseline.sql:71-96](<仓根>/migrations/0001_baseline.sql#L71-L96)）。
② 后果：从 GitHub 拉取的插件许可证无从记录、无从展示、无从在收录时校验；一旦有用户在站内被引导安装 GPL 插件并把项目再分发，责任链断在站方。方案对此完全没提。
③ 建议：若本方案的落点包含平台侧，必须在 `repositories`/`plugins` 增加许可证快照字段（含 SPDX 标识、来源 URL、抓取时间、是否可识别），并在收录流程里对"无许可证"与"禁止再分发"的仓库给显式标记；若本方案刻意只覆盖本仓自身依赖，也应在方案里**写明范围排除**，否则评审会误以为平台合规已被覆盖。

**12. 与现有 CI 契约不兼容（第 19 行 vs ci.yml）**
① `[.github/workflows/ci.yml](<仓根>/.github/workflows/ci.yml)` 只有一个 `test` job，node 版本 24（ci.yml:22），步骤带中文 name、且 README 式的注释解释了每一步为什么存在（ci.yml:3-5、17-19）。方案没有提到放在哪一步、跑在哪个 node、失败时是否阻断、是否需要 `npm ci --include=dev`。
② 后果：实现者会自由发挥；最坏情况是扫描需要一个能跑 `node_modules` 的步骤，却被塞进 `npm ci` 之前的 early step 而静默失败（license-checker 找不到包时会返回空列表并 exit 0，看起来像"全绿"）。
③ 建议：方案里显式写出——放在 `npm ci` *之后*、`typecheck` 之前/之后；与 `cache: npm`（ci.yml:23）一致；失败视为阻断；并对"扫描到的包数为 0"单独判定为失败（防空跑冒充通过）。

---

## P2：可维护性/表述（3 条）

**13. 方案没有定义"合规"的判定边界，也没写失败后的输出契约（全文）**
① 全文 23 行，只写了做什么，没写"通过/不通过"如何判定、输出什么、谁能豁免。
③ 建议：补一节"判定与输出"，至少包含：白名单及其 SPDX 语义、例外清单文件、失败输出的最小信息（包名、版本、许可证、锁定文件行号）。

**14. "防止引入 GPL 代码"（第 23 行）的表述过宽**
① GPL 不等于禁止：内核态的 LGPL 动态链接在特定条件下可用；单纯在 CI 里以工具形式使用 GPL 程序（不随产物分发）通常不产生分发义务。
② 后果：这句预期会被直接实现成"见到 GPL 就红"，把第 5 条的假阳性问题放大。**该说的其实不是"禁止 GPL"，而是"禁止把 GPL 代码分发进我们的产物"**。
③ 建议：把第 23 行改成"防止 GPL/AGPL 类许可证进入**对外分发物**"，并把"是否分发"作为判定的核心输入（这也是第 1、3 条的同一个根因）。

**15. 缺少数据留存与审计（全文）**
① 方案没说要保存扫描结果。
② 后果：出问题时无法回答"上一个版本用的某依赖当时是什么许可证"。而本仓已有 `operation_records`（[migrations/0001_baseline.sql:411](<仓根>/migrations/0001_baseline.sql#L411)）与 `background_tasks`(:345) 这类留痕表，属于同类需求的既有模式。
③ 建议：每次扫描结果落成可 diff 的 JSON 产物（随构建归档），让"许可证变了"这件事在历史里可查，而不是只有一次性的红绿灯。

---

## 一句话
方案**方向对、口径错**：它把"许可证合规"当成"白名单字符串匹配"，而 ①判定对象应从 package.json 挪到发布物（Docker 镜像 + 基础镜像 + npm 生产树）、②判定时机应从 CI 单点挪到 PR/定时/发布三层、③判定逻辑应从"是否在名单里"挪到"分发时义务是否被履行 + baseline 差异"。按当前写法落地，唯一可预见的结果是：第一周全红，第二周被禁用。
