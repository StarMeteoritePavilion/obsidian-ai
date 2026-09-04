---
title: 大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练
source: https://www.bilibili.com/video/BV1sJCnBGESj
author: 唐国梁Tommy
published: 2025-11-12
ingested: 2026-09-03
updated: 2026-09-04
tags:
  - AI
  - 模型原理
  - 大语言模型
  - Kimi-K2-Thinking
  - MoE
  - Agent
  - 资料摘要
---

# 大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练

原始资料：[[raw/sources/模型原理/大语言模型/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent|Kimi K2 Thinking 深度解析]]

## 核心结论

Kimi K2 Thinking 把 1.04T 总参数、32B 激活参数的 MoE 基础模型，与 Agent SFT、强化学习、原生 INT4 和 Test-Time Scaling 组合为一条完整技术链。模型共有 384 个专家，每个 Token 路由至 8 个专家和 1 个共享专家；第 1 个 Transformer Block 使用 Dense FFN，从第 2 个 Block 开始使用 MoE。模型采用 160K 词表、MLA 与资料所述 256K 上下文。

资料还将其与 DeepSeek R1 作结构对照：Kimi K2 Thinking 与 DeepSeek R1 的注意力头数分别为 64 与 128，MoE 专家数分别为 384 与 256，激活参数分别为 32B 与 37B；非 MoE 层分别为第 1 个 Block 与前 3 个 Block。这些数字对应资料采用的具体模型版本，不构成所有版本的固定差异。

## 预训练与数据效率

Muon 在大规模训练中可能引发 Attention Logit 爆炸。MuonClip 将 Muon、Weight Decay、Consistent RMS Matching 与 QK-Clip 结合；QK-Clip 在权重更新后按注意力头监控最大 Logit，并以 $\tau=100$ 为阈值缩放相关 Query/Key 投影权重。资料称 Kimi K2 据此完成 15.5T Token 预训练且没有 Loss Spike。

Data Rephrasing 通过多样化 Prompt、分块自回归改写和 Fidelity Verification 提高 Token 效率。资料引用的早期 K2 Checkpoint 在 SimpleQA 上，原文训练 10 个 Epoch 为 23.76，改写 1 次并训练 10 个 Epoch 为 27.39，改写 10 次且各训练 1 个 Epoch 为 28.94；结果只适用于对应实验。

## Agent 后训练与部署

SFT 数据合成依次生成 Tool Spec、Agent 与任务、交互轨迹。工具库包括 3,000 多个真实工具和 20,000 多个合成工具，User Agent、Tool Simulator 与 Judge Agent 共同生成并筛选成功轨迹，编码和软件工程任务还结合真实执行沙箱。

强化学习结合 Verifiable Rewards“Gym”（RLVR）与 Self-Critique Rubric Reward，并使用 Budget Control、PTX Loss 和 Temperature Decay。前者覆盖数学、代码、软件工程与忠实性，后者由 Actor 生成回答、Critic 按 Rubric 排序，再用可验证反馈持续校准。

MoE 组件通过 QAT 获得原生 Weight-only INT4。资料称生成速度提高约 2 倍、显存下降且性能接近无损。Kimi K2 Thinking 还通过 Test-Time Scaling 增加推理预算，并在对应设置中支持 200～300 次连续工具调用；这些收益不构成其他硬件、环境和任务的保证。

## 评测边界

Kimi K2 Thinking 在 HLE 工具设置、BrowseComp、Seal-0 和 SciCode 等项目表现较强，但并非全面领先：BrowseComp-ZH 的 62.3 略低于 GPT-5 的 63.0，SWE-bench Verified 的 71.3 也低于资料所列 GPT-5 与 Claude Sonnet 4.5。数字来自相应工具、预算和评测协议，不能合并为模型整体能力排名。

作者的 API 演示还显示，已配置的 Kimi K2 Thinking 只自称属于 Kimi 系列，没有准确报告具体模型版本。模型自述因此不能替代调用端记录的模型标识。

## 关联

- 后训练综合：[[wiki/syntheses/大模型后训练：从模仿到行为选择]]
- 推理表示与执行：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- Token 与长上下文成本：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
