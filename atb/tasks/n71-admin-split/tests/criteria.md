# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块

- [ ] **两个应用连同一个 SQLite 文件会撞写锁**
  项目的 db.ts 用 WAL + synchronous=FULL + busy_timeout=5000。
  两个应用各自一个连接 → **写锁竞争翻倍**，且 SQLITE_BUSY 会互相影响。

- [ ] **后台应用自己实现登录会绕过会话体系**
  项目的会话是 user_sessions + __Host- cookie（session-cookie.ts）。
  后台另起一套登录 = **两套会话**，管理员在主站与后台的登录状态不同步。

- [ ] **users 表的两列版本号会被并发覆盖**
  users 有 management_version 与 account_status_version（CAS 用）。
  两个应用同时改同一用户会**互相覆盖**——CAS 需要读版本号，跨应用无法保证。

### L4 系统性

- [ ] **主站的 assertCanAdmin 仍会放行后台迁走的权限**
  主站 100+ 路由的 assertCanAdmin 依赖 users.role。
  后台迁走后，主站的管理路由**仍然存在**——除非全部删除，否则是**两条管理入口**。

- [ ] **审计会分裂成两份**
  operation_records 是唯一审计载体。两个应用都写它，
  但 request_id、actor 的来源不同 → **事后无法统一追溯**。

- [ ] **备份面翻倍**
  项目每日备份 scripts/backup.mjs 打整个 .data 目录。
  两个应用若各有数据目录，备份脚本需要改；若共用，则**没有隔离**。

- [ ] **限流是 per-key 的，两个应用各自计数**
  rate_limiter_flexible 按 key 计数。两个应用各自限流 = **额度翻倍**。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
