---
title: 大模型后训练：RLM Harness 组合泛化
source: https://www.bilibili.com/video/BV1EGuZ6XEQC
author: 唐国梁Tommy
published: 2026-08-10
ingested: 2026-09-03
updated: 2026-09-03
tags:
  - AI
  - 模型工程
  - 大模型后训练
  - Agent
  - Harness
  - RLM
  - 资料摘要
---

# 大模型后训练：RLM Harness 组合泛化

原始资料：[[raw/sources/模型工程/后训练/RLM Harness：短任务训练如何迁移到长任务与新领域|RLM Harness：短任务训练如何迁移到长任务与新领域]]

## 核心结论

《Language model harnesses are compositional generalizers》把强化学习对象从孤立模型扩展为模型与 Harness 组成的系统。RLM Harness 通过 Context Offloading 和 Programmatic Subcalls，使完整任务可以分布外，但每次模型调用保持局部分布内；模型由此学习可组合的分解骨架，而不是只记住具体任务。

## 泛化机制

原始材料保存在代码环境变量中，主模型只通过少量探针查看内容；子模型和工具被包装为函数，完整返回继续保存在变量中。主 Context 主要包含代码、局部观察和分解逻辑。解法结构相同的网页检索、数据统计或跨领域任务因此可以折叠为相近轨迹，形成任务等价类。

## 实验与成本

30B 模型只在短任务上训练，评测长度扩大 8～32 倍，最长达到 2M Token。六项基准中的四项追平或超过以 GPT-5.5 为底座、未经该训练的 RLM 系统；直接训练对照组虽然取得更高训练奖励，长任务和陌生领域的迁移却很弱。

RLM 单样本训练耗时是直接训练的 1.5～3.0 倍，但主 Context 保持较短，额外成本更接近固定倍数。收益仍取决于任务是否能够干净分解，以及强化学习是否找到通用分解而非短任务捷径。

## 边界

实验任务主要是适合切块的检索、图遍历和聚合，只验证了一个 30B 底座。资料关于 Harness 泛化的结论不能直接外推到子问题高度耦合的任务、其他模型规模或所有 Agent 系统。

## 关联

- 后训练综合：[[wiki/syntheses/大模型后训练：从模仿到行为选择]]
- Context 治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Loop 与 Harness：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- Harness 定义边界：[[wiki/sources/驾驭工程：系列完结，下一步该往哪走？]]
