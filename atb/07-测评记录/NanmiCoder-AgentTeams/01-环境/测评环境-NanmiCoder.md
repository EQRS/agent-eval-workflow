# 测评环境快照 · NanmiCoder Agent Teams

> 采集时间：2026-11-17（重启后）

## 一、运行环境

| 项 | 值 |
|---|---|
| 操作系统 | win32 10.0.26200 x64 |
| CPU | 32 核 AMD Ryzen 9 8940HX with Radeon Graphics |
| 内存 | 15.2 GB |
| Node | v24.21.0 |
| **DSH** | **0.2.2-rc.1** |
| Profile | desktop |

## 二、被测对象

| 项 | 值 |
|---|---|
| **插件** | **@nanmicoder/dsh-agent-teams@0.1.30** |
| 安装位置 | `~/.dsh/profiles/desktop/node_modules/` |
| **工具命名** | **`agent_teams_*`（14 个）** |
| 状态目录 | `.agent-teams/<teamId>/` |
| memberProvider | spawn |

### 2.1 工具清单（实测可用）

| # | 工具 | 用途 |
|---|---|---|
| 1 | `agent_teams_create` | 创建团队 |
| 2 | `agent_teams_status` | 查团队状态 |
| 3 | `agent_teams_add_member` | 加成员 |
| 4 | `agent_teams_create_task` | 建任务 |
| 5 | `agent_teams_send_message` | 发消息 |
| 6 | `agent_teams_claim_task` | 认领任务 |
| 7 | `agent_teams_update_task` | 更新任务 |
| 8 | `agent_teams_delete` | 删除团队 |
| 9 | `agent_teams_approve` | 批准计划 |
| 10 | `agent_teams_edit_plan` | 编辑计划 |
| 11 | `agent_teams_reassign_task` | 重新指派 |
| 12 | `agent_teams_remove_member` | 移除成员 |
| 13 | `agent_teams_amend_task` | 修改契约 |
| 14 | `agent_teams_resume` | 恢复团队 |

## 三、被测项目

| 项 | 值 |
|---|---|
| 项目 | dsh-plugin-review@0.1.0-slice |
| 框架 | Next.js 16.4.0 |
| 数据库 | better-sqlite3 13.0.3 |

## 四、测评题库

| 项 | 值 |
|---|---|
| 题库 | `atb/tasks/`（80 题） |
| **本轮题** | **N-46（归属争议申诉）** —— 与官方同题 |
| 判据 | v3 |

## 五、与官方的关键差异

| 项 | 官方 | NanmiCoder |
|---|---|---|
| 工具命名 | `spawn_teammate` / `team_task_*` | **`agent_teams_*`** |
| 成员上限 | 8 | **待测** |
| 清理机制 | 无 | **有 `agent_teams_remove_member`** |
| 团队持久化 | 内存 | **`.agent-teams/` 落盘** |