# M7「多提供商登录」方案 · 技术评审

> 评审对象：`atb/tasks/m7-multi-provider-login/environment/proposal.md`（51 行）
> 事实基线：`migrations/0001_baseline.sql`、`migrations/0003_auth_media.sql`、`src/server/modules/identity/{service,oauth-challenges,admin,types}.ts`、`src/server/adapters/login/index.ts`、`src/app/api/auth/github/{start,callback}/route.ts`、`src/server/modules/roles/ownership.ts`、`src/server/platform/session-cookie.ts`

## ⚠️ 0. 先更正一处任务背景，否则评审会被带到错误的方向

任务背景写的是「本项目 `external_identities` 表支持 github 和 gitee 两个 provider」。**这条前提是错的，我按仓库实况核对了：**

- `migrations/0001_baseline.sql:38`：`provider TEXT NOT NULL CHECK (provider IN ('github'))` —— **只有 github**，没有 gitee。
- 同一段还有 `migrations/0001_baseline.sql:46`：`UNIQUE (user_id, provider)` —— **一个用户在同一 provider 下只能有一条身份**。
- 全仓对 gitee 的引用为 0（`src/` 下 grep `gitee` 无命中）；登录适配器只有 `'github' | 'test'` 两种（`src/server/adapters/login/index.ts:25`）。
- 另有一处同名的 provider 枚举：`migrations/0003_auth_media.sql:21` 的 `oauth_login_challenges.provider` 也是 `CHECK (provider IN ('github'))`。

**这个更正不是抠字眼，它直接决定工作量与风险**：方案第 22-24 行的「迁移历史数据」在当前 schema 下**一行都插不进去**（见 §2），而如果你相信「已经支持两个 provider」，就会以为只需要放开 CHECK。

---

## 0. 结论

**不建议按现方案实施。** 三个「必须改，否则要么不能上线、要么上线即漏洞」的点：

1. **数据迁移与库内约束冲突，执行到一半会失败**（§2）：`users` 表**根本没有 `github_id` 列**（`0001_baseline.sql:22-33`），方案的 `SELECT id, 'github', github_id ... FROM users` 会直接报错；即便有，`user_identities` 少了 `provider_username` 等列，与既有 `external_identities` 的必备列也不一致。
2. **重复造了一张已经存在的表**（§1）：本仓早已有 `external_identities`（`0001:35-47`），且**它就是**方案想要的东西（`UNIQUE(provider, provider_user_id)` + `UNIQUE(user_id, provider)`），还多带 `provider_username` / `profile_url` / `avatar_url` / `synced_at`。正确的做法是**放开那一处 CHECK**，不是新建 `user_identities`。
3. **回调链路的安全骨架全部丢失**（§3/§4/§5）：方案第 35-40 行的 5 步回调里**没有 state 校验、没有浏览器绑定、没有 PKCE**，而同目录的现有实现（`callback/route.ts:56-71`）这三样都在；照方案写就是**登录 CSRF + 授权码注入**，而且**token 与 code 会被写进 URL / 日志**（§6）。

---

## 1. 「新建 user_identities」是重复造表，且设计比现有的弱 —— proposal 第 8–20 行

**① 问题**：方案第 13-20 行新建 `user_identities`，注释写「新增身份绑定表」。

**② 后果（对照既有表）**：

| 能力 | 方案 `user_identities` | 既有 `external_identities`（`0001_baseline.sql:35-47`） |
|---|---|---|
| 同 provider 唯一身份 | `PRIMARY KEY (provider, provider_user_id)` ✅ | `UNIQUE (provider, provider_user_id)`（:45） ✅ |
| 一人一 provider 一行 | ❌ 没有 | `UNIQUE (user_id, provider)`（:46） ✅ |
| 展示用用户名 | ❌ 没有 | `provider_username`（:40） ✅ |
| 主页 / 头像 | ❌ 没有 | `profile_url` / `avatar_url`（:41-42） ✅ |
| 同步时间 | ❌ 没有 | `synced_at`（:44） ✅ |
| 外键 | `REFERENCES users(id)` **无 ON DELETE 行为**（默认 NO ACTION） | `ON UPDATE RESTRICT ON DELETE RESTRICT`（:37） |
| 枚举约束 | provider 是**自由文本** | `CHECK (provider IN ('github'))`（:38） |

- 新建表之后，站点将**同时存在两份身份数据**：`external_identities` 仍被登录主流程写（`identity/service.ts:191-206`），`user_identities` 只有新代码写。两处一旦不一致，**「这个 GitHub 账号是哪个用户」会有两个答案**。
- 更实际的一条：`external_identities.provider_username` 被**开发者身份自动匹配**依赖（`roles/ownership.ts:115-127` 的 `isGithubOwnerMatch`：`lower(ei.provider_username) = lower(r.owner_login)`）。新表不带 username，走新表登录的用户**永远无法自动匹配到自己的插件**——表现为「我是仓库作者，但系统说我不是」。

**③ 建议**：**删掉 `user_identities`，复用 `external_identities`**，只做两件事：
1. `migrations/0043` 放开 provider 枚举（见 §2 的锁表代价）；
2. 让 `signInWithGithub`（`identity/service.ts:133-215`）泛化为 `signInWithIdentity(provider, identity)`。

---

## 2. 迁移 SQL 在当前 schema 下无法执行，且「放开 CHECK」的代价没算 —— proposal 第 9–25、45–46 行

**① 问题**：方案第 22-24 行：

```sql
INSERT INTO user_identities (user_id, provider, provider_user_id, created_at)
SELECT id, 'github', github_id, created_at FROM users WHERE github_id IS NOT NULL;
```

**② 后果（核实依据）**：
- **`users` 表没有 `github_id` 列**：`0001_baseline.sql:22-33` 的全部列是 `id / nickname / avatar_url / role / account_status / management_version / account_status_version / created_at / updated_at / last_login_at`。方案第 24 行的 `SELECT ... github_id ... FROM users` 会直接抛 `no such column: github_id`，**迁移在第一个文件就失败**。
- 历史数据真正的来源是 `external_identities` 本身（`provider_user_id` 在 :39）。所以第 22-24 行**既不该跑，也不需要跑**——迁移不是「搬数据」，而是「改约束」。
- **放开 provider 枚举必须重建表**：SQLite 改不了列上的 CHECK。`external_identities` 被 `plugins`→`repositories` 之外的多处读取，且 `user_sessions` 等表有外键；重建要 `PRAGMA foreign_keys = OFF` + 自带 `BEGIN/COMMIT`，即必须在迁移文件头写 `-- dsh:migration-transaction=off`（`src/server/platform/migrate.ts:30-49` 的约定；`:97-115` 说明执行器**不会替你回滚**，中途失败会留半成品并让下次启动炸）。**方案第 45-46 行「直接执行上述 SQL」完全没提这件事。**
- 同一轮还要改 `oauth_login_challenges.provider`（`0003_auth_media.sql:21` 也是 `CHECK (provider IN ('github'))`）——**两张表都要重建**。方案一行没提，而 `oauth-challenges.ts:157` 当前是**硬编码** `VALUES (?, 'github', ...)`，泛化时很容易漏改。

**③ 建议**
- 迁移只做约束变更：`0043` 删掉 `user_identities` 建表与那条 INSERT，改为重建 `external_identities` 与 `oauth_login_challenges` 两张表、放开 provider 枚举；文件头带 `dsh:migration-transaction=off` 并自带事务。
- 迁移**前**必须先备份（`scripts/backup.mjs`），并在方案里写明回滚口径。
- 新增一条回归：迁移后 `SELECT DISTINCT provider FROM external_identities` 与历史一致（迁移不能丢行）。

---

## 3. provider 来自 URL 并决定请求域名 → SSRF / 任意出网 —— proposal 第 27–40 行

**① 问题**：方案第 30-33 行把路由改成 `GET /api/auth/{provider}/start|callback`，第 36-38 行「调对应 provider 的 API 换 access_token / 拉用户信息」。**这等于让 URL 里的字符串决定服务端去访问哪个域名。**

**② 后果（核实依据）**：
- 现有实现把域名写死在适配器里：`adapters/login/index.ts:102`（`https://github.com/login/oauth/authorize`）、`:117`（`.../access_token`）、`:160`（`https://api.github.com/user`）。**这是刻意的**——文件头铁律第 2 条（`:8-12`）写的是「出网失败/限流 → `provider_unavailable`，**绝不返回伪造身份**」。
- 一旦 provider 变成路径参数，攻击者可以请求 `/api/auth/<任意串>/callback?code=...`。若实现成「`https://{provider}.com`」或「按 provider 查一张表但表里有可注入的 URL」，就变成**服务端向任意域名发起带 `client_secret` 的 POST**（token 交换请求体里含 `client_secret`，见 `index.ts:125-131`）——**凭据外泄**；或对内网地址发起请求，即 **SSRF**。
- 还有一个不那么显眼但同样致命的：**回调地址（`redirect_uri`）必须与该 provider 注册时登记的完全一致**。方案没有说每个 provider 的 `redirect_uri` 怎么来；如果拼接错，OAuth 服务端会拒绝，表现为「登录永远失败」。

**③ 建议**
- **provider 必须是白名单枚举**，且**路由只接受枚举内的值**，非法值直接 404（不落到「未知 provider」分支去做任何网络调用）。
- 每个 provider 的 `authorize_url` / `token_url` / `user_url` / `redirect_uri` / `client_id` / `client_secret` **全部来自服务端配置**（对齐 `readGithubConfig` 的写法，`adapters/login/index.ts:34-40`），**绝不从请求里取任何 URL 片段**。
- 出网请求沿用 `fetchWithTimeout`（`index.ts:58-66`）与「失败不伪造身份」的铁律。

---

## 4. 缺 state / PKCE / 浏览器绑定 → 登录 CSRF + 授权码注入 —— proposal 第 35–40 行

**① 问题**：方案第 35-40 行的回调只有 5 步：取 code → 换 token → 拉用户 → 查表 → （找不到就）建用户。**没有任何一次性挑战校验。**

**② 后果（核实依据：本仓已经把该做的都做了）**：
- 现有回调的顺序（`src/app/api/auth/github/callback/route.ts:6-11` 的头注释）：**① 读浏览器绑定 cookie 并与挑战里的摘要比对；② 用 state 找到挑战、核验未过期未消费并原子消费；③ 用 code + verifier 换身份；④ 建会话**。`callback/route.ts:56-71` 是逐行实现；挑战服务见 `oauth-challenges.ts:139-222`。
- 方案砍掉这三层后，**登录 CSRF（login CSRF）成立**：攻击者用自己的账号发起授权、拿到回调 URL、诱导受害者点击，受害者的浏览器就会以攻击者身份登录——之后受害者的所有操作（发评测、认领插件、举报）都记在攻击者名下，攻击者事后能看到这些内容。反过来，攻击者也可以把受害者的授权码塞进自己的会话，完成**账户接管**。
- PKCE 的缺失还有一个具体后果：`oauth-challenges.ts:44-48` 的 `codeChallengeFromVerifier` 与回调里的 `codeVerifier` 是唯一让「授权码只能被发起方兑换」的机制。没有它，被截获的 `code`（例如通过 Referer、日志、浏览器历史）可以被**任何一方**兑换成 token。

**③ 建议**：**回调的骨架原样保留**，只把 `createGithubLoginAdapter()` 换成「按 provider 取得适配器」。具体：
- `oauth-challenges.create()` 已经会存 `provider` 列（`oauth-challenges.ts:154-158`），把它从硬编码改成参数；`consume()` 的返回里带上 provider，**并校验它与 URL 里的 provider 一致**（否则「A provider 的 state 配 B provider 的 code」这种混搭会过）。
- start 路由同样按 provider 生成挑战并写绑定 cookie（`start/route.ts:31-44`）。

---

## 5. `email` 用于建号/关联 = 账户接管 + 信息缺失 —— proposal 第 17、40 行

**① 问题**：方案第 17 行给了 `email` 列，第 40 行「如果找不到，创建一个新用户」。既有表与代码**从头到尾没有 email**（`external_identities` 无该列，`signInWithGithub` 也不接收 email）。

**② 后果**：
- 一旦有人用 email 做「已存在则关联」的判断：**Gitee / GitLab 这类提供商允许用户把邮箱设成任意值（未验证前即可读）**，攻击者把自己的邮箱改成受害者地址 → 下次受害者用 GitHub 登录时被**关联到攻击者的账号**，或攻击者用该邮箱登录直接进入受害者账号。这是多提供商登录里最经典的**账户接管**路径。
- 本仓**已经有正确的口径**：身份唯一性只认**稳定账户 ID**——`adapters/login/index.ts:11` 铁律第 3 条：「只使用 GitHub 稳定账户 ID（数字 id）作为身份标识，用户名可改，不能当身份」；`identity/service.ts:134-139` 在校验 providerUserId 时就写明「GitHub 稳定 ID 是身份唯一性的根」。**方案没有提到这条规矩，还引入了一个弱标识（email）**。
- 顺带：方案第 17 行的 `email TEXT` 没有 `CHECK`、没有唯一约束——即使只做展示也会造成「同一个人两个邮箱」的两份记录。

**③ 建议**：**删掉 email 列**。身份只认 `(provider, provider_user_id)`。若要支持「用 email 找回」，作为**独立课题**单独立项，并且必须满足「provider 明确声明 email_verified + 与既有已验证邮箱一致 + 需要额外验证步骤」三个条件，**不能**在登录回调里出现。

---

## 6. 会话与凭据处理：方案第 42-43 行的「沿用现有机制」掩盖了两个具体风险

**① 问题**：方案第 43 行「沿用现有的 session cookie 机制，无改动」。

**② 后果**：
- **正面结论先说**：真正的 Cookie 策略确实是 provider 无关的，`session-cookie.ts:45-56` 的 `resolveCookieMode` 只看环境，`writeSessionCookie`（`:92-101`）固定 `httpOnly + secure + sameSite:'lax' + path:'/'`。**这一条方案说得对**，不需要改。
- 但「无改动」漏掉了一个必须决定的事：**一个用户绑多个提供商，登录后是同一个 `user_id` 吗？** 方案第 51 行「一个用户可以绑定多个提供商」需要一条绑定入口（已登录用户主动绑定），**否则只能靠 email 猜**（§5 的接管路径）。方案没有绑定流程，也没有「解绑」——解绑涉及「这个账号还有没有别的登录方式」，即**不能把用户锁在门外**。
- 凭据面：token 交换请求体里含 `client_secret`（`adapters/login/index.ts:125-131`）。方案的 provider 泛化会让**每种 provider 各有一份 client_secret**；如果按「统一从 `PROVIDER_*` 前缀批量读」实现，一个 provider 的配置错误会连带其它 provider 一起不可用。建议每个 provider 独立配置、独立判定 `isConfigured()`（对齐 `index.ts:81-83`）。

**③ 建议**：方案补一节「绑定/解绑」：已登录用户在设置页发起绑定 → 走同一套挑战流程 → 成功后写 `external_identities` 一行（`UNIQUE(user_id, provider)` 保证不会重复绑）；解绑必须校验「至少还剩一种登录方式」。

---

## 7. `users` 与迁移带来的连带影响（方案未提，但会直接影响上线）

- **`admin.ts:151-153`**：`LEFT JOIN external_identities ei ON ei.user_id = u.id AND ei.provider = 'github'` —— 后台用户列表把 `provider_username` 当作 `github_login`。多 provider 后，这里要么改成「显示第一个 provider」，要么改成多值展示，否则**后台看到的「GitHub 登录名」会莫名其妙变空**。
- **`me/settings/page.tsx:72`、`onboarding/page.tsx:62`**：同样按 `provider` 取一行。多 provider 后需要决定展示哪一个。
- **`roles/ownership.ts:123`**：`lower(ei.provider_username) = lower(r.owner_login)` —— 这是开发者身份自动匹配。**provider 泛化后这条 SQL 会匹配到 gitee 用户名**，而 `repositories.owner_login` 是 **GitHub** 的（`0001:71-84` 的 `repositories` 只有 `github_repository_id`）。如果不加 `ei.provider = 'github'` 条件，**会出现「gitee 上有个同名账号」就自动拿到插件归属**——这是本次改动最容易漏掉的一条**越权**。

**建议**：`0043` 的同时把 `isGithubOwnerMatch`（`ownership.ts:115-127`）显式限定 `provider = 'github'`，并加一条回归用例：gitee 同名账号不能自动匹配 GitHub 仓库。

---

## 8. 验收：方案第 48-50 行的三条预期都不可证伪

| # | 断言 | 场景 |
|---|---|---|
| 1 | 迁移在**已有数据**的库上跑通，`external_identities` / `oauth_login_challenges` 行数不变 | 用备份副本演练 |
| 2 | `/api/auth/<任意非白名单串>/callback` → **404，且无任何出网请求** | 抓包/打桩 |
| 3 | 缺 state / 错 state / 重放同一 state / 换浏览器（无绑定 cookie）→ **全部拒绝** | 手工 4 例 |
| 4 | gitee 同名账号**不能**自动匹配 GitHub 仓库归属 | 构造同名仓库 |
| 5 | 同一 provider_user_id 并发回调两次 → 只建 1 个用户（`identity/service.ts:146-148` 的事务语义保持） | 并发脚本 |
| 6 | 已禁用账户（`account_status='disabled'`）用任一 provider 登录 → 仍然 `err.disabled()`（`identity/service.ts:165-168`） | 手工 |

## 9. 一句话给作者

**身份表已经有了（`external_identities`），挑战与会话骨架也已经有了（`oauth-challenges.ts` + `callback/route.ts`）。这次真正要改的只有三处：放开两张表的 provider 枚举、把适配器按 provider 参数化、把 `create()` 里硬编码的 `'github'` 变成参数。** 方案现在写的是「新建一张更弱的表 + 一段在库里跑不通的迁移 + 一条丢掉全部安全校验的回调」——这三样都不是增量，是替换。
