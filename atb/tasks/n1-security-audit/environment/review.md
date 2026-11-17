# 评审报告：《会话机制从 Session Cookie 改为 JWT》（tasks/n1-security-audit/environment/proposal.md）

评审基准：只依据本仓库现有代码与迁移核实，不改代码。行号均指当前工作区文件。

---

## P0

### 1. 【方案 §2+§3+§4】没有会话撤销：登出、禁用、撤权全部失效
**问题**：方案把"校验"压缩成一次 `jwt.verify`，**完全绕开 `user_sessions` 表**，且没有任何吊销名单/jti 概念；§1 有效期 7 天。
**后果**（对照现有实现逐条退化）：
- `src/app/api/session/route.ts:67-80` 的 `DELETE /api/session` 现在是**服务端撤销**（`identity.revokeSession` → `UPDATE user_sessions SET revoked_at`）；改 JWT 后前端只能删 localStorage，token 在服务端仍是有效凭证 → 窃取者（XSS、公共电脑）7 天内通行。
- `src/server/modules/identity/admin.ts:565-572`：禁用账户时在**同一事务**里撤销该用户全部会话，注释写的就是"禁用 = 立刻踢下线"。JWT 无状态后这条路径**整体失效**：被禁用者继续写评测/评论 7 天（`assertCanWrite` 只看 payload 里的 accountStatus 字段，方案里连这个字段都没有）。
- `src/server/modules/identity/service.ts:258-267`：现有 `resolveSession` 逐条显式判断 revoked/expires/absolute_expires/account_status；方案把这些判断**全部丢弃**，只剩签名有效性。
**建议**：① 保留服务端会话行或引入`token_version`（`users.session_epoch`）随 `userId` 一起签发，每次校验比对；② 或者接受"短 access token(≤15min) + 服务端 refresh token（仍落 `user_sessions`，登出/禁用即撤销）"；③ 若坚持纯无状态，必须补一张 jti 黑名单表并给出清理策略——那就又回到查库，方案 §"预期-省掉每请求的会话查询"的前提不再成立。

### 2. 【方案 §2】token 存 localStorage，把 XSS 升级为会话完全失守
**问题**：为"跨请求带 Authorization 头"把凭证从 httpOnly Cookie 挪到 JS 可读的 localStorage；且**没有提 CSRF 之外的任何补偿**。
**后果**：
- 现有 `src/server/platform/session-cookie.ts:92-101` 是 httpOnly + Secure + `__Host-` 前缀 + SameSite=Lax（注释 1-21 行明确讲了这个设计是刻意的）。方案直接倒退回 OWASP 明令避免的做法：任一处 XSS 即可 `localStorage.getItem` 拿走 7 天有效期的**不可撤销** token（与 P0-1 叠加成 7 天彻底接管）。
- 站点有用户生成内容渲染链路（unified/rehype-sanitize，见 package.json），渲染面越广，这个风险越实。
- 反而引入了新的隐患面：现在没有任何 CSRF 防御代码（SameSite=Lax 隐含防御），方案未提 SameSite/CSRF token 方案。
**建议**：token 继续放 httpOnly Cookie（同源，无需 Authorization 头）；如果确实需要非 Cookie 客户端，短期 access token 也要做到 5-15 分钟 + 可撤销，且不能落 localStorage。

### 3. 【方案 §4】角色变更对已签发 token 不生效 —— 方案自己写错了
**问题**：§4 原文"用户角色改了之后，新 token 会包含新角色"，**回避了旧 token**：payload 里的 `role` 在签发时冻结，7 天内继续使用旧权限。
**后果**：本仓库的角色模型是**实时生效**的，JWT 会把它整体推翻：
- `src/server/platform/context.ts:1-6,23-25` 顶部注释白纸黑字："角色与账户状态**每次请求从数据库读取**，不缓存在会话里：撤销管理员后，其普通登录会话不因此获得旧管理权限"；`isAdmin` 直接读 `ctx.role`/`ctx.accountStatus`。
- `src/server/modules/roles/service.ts:738-768`（`revokeRole`）与 `admin.ts:532-563`：撤销管理员/禁用账户后，下个请求就该 403；JWT 下**多出最长 7 天的提权窗口**，等于本仓库已写明的安全不变量被删掉。
- 所有基于 `assertCanAdmin` 的管理操作（审核、删分类 `catalog/service.ts:1153`、禁用账户）都会接受"已失效管理员"的旧 token。
**建议**：role 不进 payload，或接受"每次查一次 users 表"（那 `assertCanWrite` 的 account_status 也需要库）；也可两者都不进，`role` 只在鉴权层即时解析。方案必须明确写"权限类字段不得缓存进 token"。

### 4. 【方案 §3】校验代码缺两条硬性判断，且把代码缺陷带进中间件
**问题**：
```ts
const payload = jwt.verify(token, process.env.JWT_SECRET);
ctx.userId = payload.userId;
ctx.role = payload.role;
```
- 没有校验 `payload.alg`/显式指定 `algorithms: ['HS256']`（算法混淆/alg=none 经典坑）；
- 没有处理 `JWT_SECRET` **未配置**的情况：`process.env.JWT_SECRET` 为 `undefined` 时 `jwt.sign/verify` 会抛错或退化为弱密钥，而当前 `.env.example:43-45` 只有 `SESSION_SECRET`（明确写"生产必须设置，否则拒绝登录"），**新变量没有任何启动期自检**；
- `jwt.verify` 抛异常时的映射没有说明——本项目 `context.ts:31-33` 强调必须抛 `DomainError` 才能映射 401/403，裸 Error 会变 500（`assertCanWrite` 注释同理）；中间件里漏 try/catch 会让所有带坏 token 的请求 500。
- `role` 未做白名单校验（任意字符串进 `ctx.role`），配合 `isAdmin` 的 `role === 'admin'` 比较，一旦签名密钥泄露可直接铸 admin。
**建议**：`verify(token, secret, { algorithms: ['HS256'] })`；启动期 assert secret 存在且 ≥32 字节；payload 用 zod 校验 `{userId: string, role: 'user'|'admin'}`；verify 失败统一 `err.unauthenticated()`。

### 5. 【方案 §5 与部署事实矛盾】双进程部署让"新旧密钥并存"不成立
**问题**：§5 只说"新旧密钥并存一段时间"，完全没提**谁认哪把钥匙**。本仓库是三容器部署：`docker/compose.yaml` 的 `migrate`(27)、`web`(44)、`worker`(78) 共用镜像但**各自独立读 `env_file: ['.env']`**；库是 `/app/.data/app.sqlite` 单文件跨容器共享。
**后果**：
- 轮换时要同时改 N 个容器环境变量并重启；中间任何时刻"只重启了一半"，就是一批用户随机 401（web 新钥签、另一实例旧钥验）。方案未给"先加新钥（双钥验签）→ 再切签发 → 最后删旧钥"的三阶段流程，实际很容易做成一步切换。
- 若用 `JWT_SECRET` 单值环境变量，**根本表达不了"新旧并存"**，需要 `JWT_SECRETS` 数组/带 kid 的密钥集。
- 反过来，若真按"新旧并存"实现双钥同时验签，**又削弱了 P0-1**：泄露的旧钥在并存窗口内仍可验，等于给了额外攻击面；方案里没有任何"旧钥何时作废"的时钟。
- 每季度强制轮换还会让所有人 7 天 token 在轮换点集中失效（若旧钥被移除），方案未提用户体验与告警。
**建议**：用 `kid` + 密钥集（`JWT_KEYS`：kid→secret），验签按 kid 选钥、签发只用当前 kid；在文档里写明三阶段流程与"旧钥作废时间点"；把密钥下发纳入部署清单（compose 的 web/worker 同时更新），或改为从库里读密钥集（`configuration_revisions` 已有版本化配置表可用）。

---

## P1

### 6. 【方案 §"背景"与"预期"】收益论证不成立，且与现有设计冲突
**问题**："每次请求查一次库"被当作成本，实际现有实现已经是**低频写**：`identity/service.ts:294-304` 规定续期间隔 24h、`last_seen_at` 按 `activityWriteIntervalSeconds` 节流；读侧是 `token_digest` 唯一索引的**单行等值查询**（`0001_baseline.sql:50-65`：`UNIQUE(token_digest)`），且 `resolveSession` 注释明确"角色与账户状态每次从库里读，不缓存在会话行里"。better-sqlite3 本地同步读，成本是微秒级。
**后果**：为一个几乎不存在的性能问题，牺牲 P0-1/2/3 三条安全不变量；而为了补回撤销能力又得查库/查名单，净收益可能为负。
**建议**：如果目标是"减少查询"，先在真实负载上测（`scripts` 下有性能脚本），并给出量化对比；否则方案结论应为"不改造"。

### 7. 【方案全文】迁移与回滚路径缺失
**问题**：没有迁移计划：`REQUIRED_TABLES` 里 `user_sessions` 是**启动自检必需表**（`src/server/platform/migrate.ts:145`），文档迁移也没说这张表留不留；没有灰度/双写期；没有"线上用户已在 Cookie 里持有旧 token"的平滑过渡方案（用户会被强制登出一次）。
**后果**：上线即全体掉线；若同时删表则启动自检直接失败（`missingTables` 非空），web/worker 起不来；想回滚时旧会话已作废。
**建议**：写明"保留 `user_sessions`（refresh token 仍用它）/ 或同时接受 Cookie 与 Bearer（过渡期 N 天）/ 或强制重登并公告"三选一；迁移加在 `migrations/` 并给出 `REQUIRED_TABLES` 的变更；明确回滚判据。

### 8. 【方案 §1】过期策略与项目既定口径不一致
**问题**：`expiresIn: '7d'` 只描述了弹性的 7 天，丢掉了现有的**双时间轴**：`session-cookie.ts:83` `MAX_AGE_SECONDS = 30*24*60*60`（绝对 30 天，注释指向"架构第 105 节"）、`user_sessions` 的 `expires_at <= absolute_expires_at` CHECK（`0001_baseline.sql:56-62`）、以及 `touchSession` 的续期语义（`identity/service.ts:306-310`）。JWT 一旦签发无法续期也无法提前失效。
**后果**：会话寿命语义回退到"签发即固定 7 天"，与架构第 105 节的"滑动 7 天 + 绝对 30 天上限"冲突；用户既不能续期（必须重登），也无法被缩短。
**建议**：明确写出 access token 短时 + refresh token 实现滑动续期，并把绝对上限规则迁移过去；否则应在方案里显式声明"有意放弃 105 节口径"并请决策者确认。

### 9. 【方案 §1】token 膨胀与 Payload 内容边界未定义
**问题**：`{ userId, role }` 之外没有约束：是否要放 `accountStatus`、`nickname`（`api/session/route.ts:33-39` 现在从 users 表读昵称）、会话 id；也没规定 PII 上限。JWT 是 base64 明文可读。
**后果**：后期"顺手把昵称/邮箱塞进去"会把 PII 暴露在 localStorage、日志（`logging` 驱动落盘 json-file）与 Sentry 类工具的 URL/头里；token 变长增加每个请求的头开销。
**建议**：方案里写明 payload 白名单（只放 userId + 不敏感的会话标识），PII 一律走 `/api/session` 查询。

### 10. 【方案 §2】改动面被低估：现有调用方全部依赖 Cookie 语义
**问题**：方案把改动写成"中间件统一校验"一句，但现状是 `currentContext()`（`src/app/api/_lib/context.ts:21-28`）经 `readSessionToken()` 从 `cookies()` 读，全站 API（含 `api/session/route.ts`、前端 `session-nav.tsx:53,95`、`comment-section.tsx:104`）都走这条路。改 Bearer 后**前后端要同步发版**，且 `fetch` 默认不跨源带 header、`no-store` 缓存语义也要复核。
**后果**：容易出现"后端已切 JWT、前端旧版本仍在读 Cookie"的窗口期，表现为全站登出；方案没给兼容层。
**建议**：在 `readSessionToken/currentContext` 处做"Cookie 优先、Bearer 兜底"的双读过渡，前端只在确认后再移除 Cookie 写入。

### 11. 【方案 §3】缺少限流/重放与审计字段
**问题**：现有鉴权异常路径在 `context.ts` 之外还有速率限制（`rate-limiter-flexible` + `rate_limiter_flexible` 表）；JWT 中间件若在 verify 失败时不明文区分原因，仍会被暴力探测。另外 `user_sessions.id` 现在作为 `sessionId` 出现在审计/上下文（`ResolvedSession`），JWT 里没有它，`touchSession`/审计关联会丢维度。
**后果**：审计记录里的"哪次会话做了什么"退化为 "userId + 时间窗"；安全事件复盘能力下降。
**建议**：payload 里带上 `sid`（会话/设备标识），日志与 `operation_records` 继续用 `sid` 关联；verify 失败对客户端统一 401，对日志记录细分原因。

---

## 结论
方案在**性能动机**上未经量化、在**安全不变量**上与本仓库三处显式设计（`context.ts:1-6` 实时读角色、`admin.ts:565-572` 禁用即踢线、`session-cookie.ts` 的 httpOnly/`__Host-`）正面冲突，且缺少迁移与密钥管理流程。**建议驳回当前形态**，若仍要做，最小可行版本是：httpOnly Cookie + 短时 access token + 服务端 refresh token（保留 `user_sessions` 与撤销路径）+ kid 密钥集 + payload 白名单。