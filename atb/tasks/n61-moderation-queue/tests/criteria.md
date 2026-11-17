# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？
  **注意**：`users.role` 目前只有 `('user','admin')`，方案**声明要新增** moderator——这属合理（有 ALTER）。
  但要判断 ALTER 的写法是否可行。

## 必须发现（L3 层）

### L3 跨模块
- [ ] **users.role 的 CHECK 无法用 ALTER 修改**
  SQLite **不支持** `ALTER TABLE ... DROP CONSTRAINT`。
  要改 CHECK 闭集必须**重建表**（建新表 → 拷数据 → 删旧表 → 改名）。
  且 `users` 被 **76 处外键引用**，重建需要 `PRAGMA foreign_keys=OFF`，
  项目有 `dsh:migration-transaction=off` 的约定（`0007` 迁移是范例）。
  §1 的两条 ALTER 语句**直接语法错误**。

- [ ] **isAdmin() 只认 role === 'admin'**
  `src/server/platform/context.ts`：
  `ctx.role === 'admin' && ctx.accountStatus === 'active'`。
  新增 moderator 后，**isAdmin 对审核员仍返回 false**——
  `decideByAdmin` 的 `if (!isAdmin(ctx)) throw err.forbidden` 会把审核员挡在门外。

- [ ] **assertCanAdmin 与 admin 路由全部失效**
  `src/app/api/admin/**` 下的路由都调 `assertCanAdmin`。
  方案说「审核员通过 /admin/submissions 处理」——**这条路走不通**。

### L4 系统性
- [ ] **改 role 闭集会波及所有 role 判断点**
  全仓多处判断 `role === 'admin'` 或 `role !== 'user'`。
  新增第三个值后，**所有「非 admin 即普通用户」的假设都会失效**。
  例如某些代码可能写 `if (role !== 'admin') { 普通用户逻辑 }`，
  moderator 会**走进普通用户分支**。

- [ ] **CallerContext 的类型需要同步扩展**
  `ctx.role` 的类型若定义为 `'user' | 'admin'`，
  新增 moderator 后类型不匹配，**编译期就会报错**。
  但若有 `as any` 或字符串比较，则**静默失效**。

- [ ] **审核员看到的敏感内容没有访问控制**
  待审内容可能是**违规内容**。方案让审核员通过 admin 页面看，
  但 admin 页面的数据范围是**全量**——审核员会看到**所有**待审内容，
  而不只是分配给自己的。

- [ ] **分配表没有唯一约束，会重复分配**
  §2 的 `review_assignments` 无 `UNIQUE(submission_id)`，
  §3 的查询也不排除已分配的 → 每 5 分钟重复插入同一批。

- [ ] **reviewers 为空时崩溃**
  §3 的 `i % reviewers.length` 在 `reviewers.length === 0` 时
  得到 `NaN`，`reviewers[NaN]` 抛 `TypeError`。

- [ ] **没有处理审核员被禁用**
  审核员 `account_status = 'disabled'` 后，
  已分配给 Ta 的任务**永远卡住**。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
