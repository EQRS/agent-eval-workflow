# 测评环境快照 · 官方 Agent Teams

> 采集时间：2026-11-17T05:30:33.439Z

## 一、运行环境

| 项 | 值 |
|---|---|
| 时区 | Asia/Shanghai |
| 平台 | win32 10.0.26200 |
| 架构 | x64 |
| CPU | 32 核 AMD Ryzen 9 8940HX with Radeon Graphics         |
| 内存 | 15.2 GB |
| Node | v24.21.0 |
| **DSH** | **0.2.2-rc.1** |

## 二、被测项目

| 项 | 值 |
|---|---|
| 项目 | dsh-plugin-review@0.1.0-slice |
| Next | 16.4.0 |
| better-sqlite3 | 13.0.3 |

## 三、被测对象

| 项 | 值 |
|---|---|
| **智能体团队** | **官方 Agent Teams（DSH 内建）** |
| 团队工具 | spawn_teammate / send_message / wait_agent / list_agents / interrupt_agent / team_task_* |
| 共享模型 | deepseek-v4.1-flash-fastest |

## 四、对照组

| 组 | 配置 | 说明 |
|---|---|---|
| **S 组** | 1 个 subagent | 单 agent 基线 |
| **T 组** | Lead + 3 teammates | 官方 Agent Teams |

## 五、测评题库

| 项 | 值 |
|---|---|
| 题库 | atb/tasks/（80 题） |
| 本轮题量 | （待定，见测评方案） |
| 判据结构 | v3（前提核实 0.5 + 预设 60% + 超纲 40%） |