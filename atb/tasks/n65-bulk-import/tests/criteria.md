# 判据清单（agent 不可见）

> **评分结构 v3**：前提核实 0.5 分 / L1-L2 20% / L3 跨模块 40% / L4 系统性 40%

## 前提核实

- [ ] 方案假设的对象是否真实存在？

## 必须发现（L3 层）

### L3 跨模块
- [ ] **users 表有多个 NOT NULL 列没赋值**
  `migrations/0001_baseline.sql` 的 `users`：
  `account_status NOT NULL`、`management_version NOT NULL`、
  `account_status_version NOT NULL`、`created_at NOT NULL`、`updated_at NOT NULL`。
  §1 只给 3 列，**必然违反 NOT NULL 约束**。

- [ ] **role 是 CHECK 闭集**
  `CHECK (role IN ('user','admin'))`——CSV 里若写 "管理员" 会违反约束。

- [ ] **导入的用户没有 GitHub 身份，无法登录**
  项目的登录是 GitHub OAuth（`external_identities` 表）。
  直接插 `users` 行的用户**没有 provider_user_id**，
  永远无法通过 OAuth 登录——这是个"僵尸账号"。

### L4 系统性
- [ ] **没有去重，重复导入会创建重复用户**
  §1 无 `ON CONFLICT`、无预查询。
  项目 `external_identities` 有 `UNIQUE(provider, provider_user_id)`，
  但 `users` 表**没有 nickname 唯一约束**。
  同一 GitHub 用户被导入两次 = 两个 user 行。

- [ ] **循环无事务，部分失败会留下半批数据**
  §1 的 for 循环无 `withTransaction`。
  第 5 行失败时前 4 个已插入，**且返回"成功 4 失败 1"**——
  管理员无法知道"这批到底要不要重跑"。

- [ ] **CSV 解析极其脆弱**
  `csv.split('\n').map(line => line.split(','))`：
  逗号在字段值里、换行符在不同操作系统（\r\n vs \n）、
  **引号包裹的字段**全部会解析错误。
  项目 `imports` 模块已有 CSV 处理逻辑，方案没有复用。

- [ ] **没有上限，一个 CSV 可以塞入 10 万行**
  在 `synchronous=FULL` 下是 10 万次 fsync。
  项目 `rate-limit.ts` 的动作枚举里**没有 import 动作**。

- [ ] **没有审计**
  批量创建用户是**高危操作**（等于批量授权）。
  项目 `operation_records` 是审计载体，方案**没有写审计**。


## L4 系统性（每项 2 分）

- [ ] **是否发现了跨模块的隐藏关联？**

## 超纲发现

- [ ] 预设之外、经核实成立的问题
