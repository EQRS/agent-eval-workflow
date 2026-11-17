# 发现 A · subagent 并发上限 18，与团队争抢

> 采集：2026-11-17

## 现象

同时启动 16 个 S 组 subagent 后，给 teammate 发消息时报错：

```
subagent limit reached (active child limit: 18);
wait for an existing child to finish or complete this work with the current agents
```

## 说明

| 项 | 结论 |
|---|---|
| **全局 subagent 上限** | **18** |
| S 组占用 | 16 |
| teammate 可用 | 2 |
| 结果 | teammate 无法并行工作 |

## 对测评流程的影响

> **S 组（16 个 subagent）与 T 组（teammate）会争抢同一个 subagent 池。**

**必须分阶段**：
1. 先跑 S 组（16 个 subagent）
2. 等 S 组完成后再跑 T 组

或者：
- 减少 S 组并发（如分 2 批，每批 8 个）
- T 组用已有 teammate（不新 spawn）

## 这是《测评流程 v1》需要补充的坑

第 9 个坑：**S 组与 T 组争抢 subagent 池，必须分阶段执行**。
