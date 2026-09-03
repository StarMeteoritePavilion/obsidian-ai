---
title: Agent 强化学习基础设施：Kimi K3 AgentENV
source: https://www.bilibili.com/video/BV1iXuH6UEd5
author: 唐国梁Tommy
published: 2026-08-06
ingested: 2026-09-03
updated: 2026-09-03
tags:
  - AI
  - 模型工程
  - 训练基础设施
  - Agent
  - 强化学习
  - AgentENV
  - 资料摘要
---

# Agent 强化学习基础设施：Kimi K3 AgentENV

原始资料：[[raw/sources/模型工程/训练基础设施/让AI学会“存档”！Kimi K3的5122万个Agent沙箱揭秘：Agent强化学习基础设施到底有多复杂？|让 AI 学会“存档”：Kimi K3 AgentENV]]

## 核心结论

AgentENV 把 Agent 操作的文件系统、进程、内存和其他操作系统状态变成可暂停、恢复、复制与定期保存的训练资源。它不保存模型内部思考，也不管理模型权重、请求状态或 KV Cache；其职责是让长时程 Agent 轨迹的外部环境能够跨等待期和训练迭代持续存在。

Kimi K3 技术报告称，训练和评估期间累计创建了 51,219,741 个沙箱，涉及 1,505,678 种镜像。这是跨时间、跨任务的累计创建量，不是同时运行的虚拟机或物理服务器数量。

## Partial Rollout 与环境生命周期

Partial Rollout 在一批轨迹完成到设定比例后先启动策略优化，把未完成轨迹暂停并排队，下一轮优先恢复。AgentENV 负责保存和恢复这些轨迹使用的外部环境，模型版本、KV Cache 和请求状态则由其他基础设施处理。

技术报告还指出，在部分工作负载中，等待模型推理最高可占沙箱生命周期的 98%。Pause and Resume 让空闲沙箱释放 CPU 和内存；Fork 为奖励评判创建不影响原轨迹的精确副本；Snapshot 为长任务提供定期故障恢复点。

## 隔离、效率与代价

传统容器依靠 Namespace 和 Cgroup 隔离进程与资源，却共享宿主机内核。Kimi 团队早期观察到意外 Agent 操作引发的 kernel panic 和 deadlock，因此 AgentENV 采用 Firecracker microVM，在独立内核隔离与高密度短生命周期之间取舍。

增量检查点只保存被修改的内存页，报告中的最低检查点和恢复延迟分别约为 133 毫秒与 49 毫秒。Copy-on-Write、页面缓存、OverlayBD、存储层共享和点对点传输进一步降低复制与镜像开销；真实工作负载中的内存超配比例最高达到 6.5 倍。最低延迟和最高超配均是对应系统条件下的边界数据，不构成其他集群的性能保证。

长轨迹跨越模型更新后会产生 Off-Policy 数据陈旧。Kimi K3 用逐 Token 正则化限制策略变化；这属于模型训练侧约束，与 AgentENV 的环境持久化互补而不互相替代。

## 与现有知识的整合

该资料补全了 Agent RL 轨迹优化的环境层：奖励、Rollout 和算法之外，训练系统还必须管理外部状态的隔离、暂停、恢复与复制。它也把 Agent Harness 中抽象的“执行环境和恢复”落到 microVM、快照和存储机制上。

AgentENV 自身不构成长期 Loop。Partial Rollout 决定训练迭代何时暂停和恢复，AgentENV 提供可恢复的环境状态；只有调度、验证、状态写回和停止条件共同存在，才形成完整闭环。

## 关联

- 后训练综合：[[wiki/syntheses/大模型后训练：从模仿到行为选择]]
- Agent Harness 综合：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- 循环工程综合：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- Agent RL 路线：[[wiki/sources/大模型后训练：强化学习如何选择反馈与算法]]
