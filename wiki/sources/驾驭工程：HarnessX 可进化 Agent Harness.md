---
title: 驾驭工程：HarnessX 可进化 Agent Harness
source: https://www.bilibili.com/video/BV1Lxj86RECd
author: 唐国梁Tommy
published: 2026-06-21
ingested: 2026-09-03
updated: 2026-09-03
tags:
  - AI
  - Agent
  - Agent Harness
  - HarnessX
  - 驾驭工程
  - 资料摘要
---

# 驾驭工程：HarnessX 可进化 Agent Harness

原始资料：[[raw/sources/应用工程/驾驭工程/Agent 的真正瓶颈不是模型，而是这层 Agent Harness｜HarnessX 全景解读|Agent 的真正瓶颈不是模型，而是这层 Agent Harness｜HarnessX 全景解读]]

## 核心结论

HarnessX 把 Prompt、工具、记忆、控制流和运行环境组成的 Agent Harness 从静态手工代码改造成可序列化、比较、替换和自动进化的一等对象。它以窄接口 Processor 和固定生命周期挂载点实现组合，以 AEGIS 根据轨迹与验证器改进外壳，再用 Cross-Harness GRPO 让模型与 Harness 共享轨迹并协同训练。

## 安全进化机制

“运行对偶”把 Harness 配置、修改、执行反馈和接受闸门分别映射为强化学习中的状态、动作、反馈和更新，因此 Reward Hacking、Catastrophic Forgetting 与 Under-exploration 也会出现在外壳进化中。

AEGIS 用 Digestor、Planner、Evolver 和 Critic 四角色流水线压缩证据、扩大探索、生成候选并审查声明；确定性闸门则拒绝让已解决任务退化的修改。语言模型拥有提议权，类型与规则拥有放行权。

## 变体与协同进化

单一 Harness 无法同时满足互相冲突的任务时，变体隔离允许系统分岔配置并按任务路由。Cross-Harness GRPO 进一步把同任务、不同 Harness 版本产生的轨迹放入同一奖励组，让外壳变化承担结构化探索，模型再内化高回报行为。

## 结果与限制

资料转述的五基准、三任务模型实验中，15 个组合有 14 个改善，平均提升 14.5%，最高提升 44%；弱模型的增益更大。四角色与单 Agent 进化器的最终准确率接近，主要优势是减少约 14% Token 并提高可审计性。

所有提升都在进化任务上测量，没有独立留出测试集；实验只覆盖离散文本动作，依赖 Opus 4.6 这类强根 Agent，协同训练还要求同时控制 Harness 与模型。因此这些结果不能直接外推为生产收益。

## 关联

- Harness 综合：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- Harness 定义演进：[[wiki/sources/驾驭工程：系列完结，下一步该往哪走？]]
- 循环工程：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- 评估工程：[[wiki/syntheses/评估工程：从通用基准到业务质量门]]
- 后训练：[[wiki/syntheses/大模型后训练：从模仿到行为选择]]
