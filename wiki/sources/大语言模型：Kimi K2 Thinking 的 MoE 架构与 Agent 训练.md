---
title: 大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练
source: https://www.bilibili.com/video/BV1sJCnBGESj
author: 唐国梁Tommy
published: 2025-11-12
ingested: 2026-09-03
updated: 2026-10-10
tags:
  - 模型专题
  - AI
  - 模型原理
  - 大语言模型
  - Kimi-K2-Thinking
  - MoE
  - Agent
  - 资料摘要
---

# 大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练

原始资料：[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent|Kimi K2 Thinking 深度解析]]

## 核心结论

Kimi K2 Thinking 的官方模型卡列出 1T 总参数、32B 激活参数、61 层且包含 1 个 Dense 层、384 个专家、每 Token 选择 8 个专家及 1 个共享专家，并采用 160K 词表、MLA 与 256K 上下文。基础模型 Kimi K2 技术报告中的 1.04T 与模型卡的 1T 属于精确值和取整值的区别。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#架构摘要|原文：官方架构摘要]]）

证据分为两组：Kimi K2 技术报告说明基础模型的预训练、数据合成和 Agent 后训练；Thinking 模型卡与发布页说明 Thinking 的配置、INT4、长程工具调用与评测。不能把基础报告中的方法自动写成 Thinking 版本新增或完整的训练配方。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#强化学习机制的证据边界|原文：版本归属]]）

## 预训练与数据效率

基础模型 Kimi K2 技术报告第 2.1 节介绍 MuonClip：组合 Muon、权重衰减、更新量 RMS 匹配与 QK-Clip；QK-Clip 读取前向计算已得到的各头最大 Logit，超过 $\tau=100$ 时缩放相关 Query／Key 权重。报告称基础模型在 15.5T Token 预训练中没有 Loss Spike；这不是任意模型采用该方法后的稳定性保证。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#基础模型如何稳定完成预训练|原文：MuonClip]]）

Data Rephrasing 通过多样化 Prompt、分块自回归改写和一致性检查提高 Token 效率。基础报告 Table 1 的早期 K2 检查点在 SimpleQA 上：原文训练 10 个 Epoch 为 23.76，改写 1 次训练 10 个 Epoch 为 27.39，改写 10 次训练 1 个 Epoch 为 28.94。报告称大规模知识语料实际最多改写两次，不能把 10 次改写当作生产配方。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#Data Rephrasing提高高质量数据利用率|原文：数据改写实验]]）

## Agent 后训练与部署

基础报告第 3.1.1 节依次建立工具规范、生成 Agent 与任务、生成并筛选轨迹。真实库来自 GitHub 上 3,000 多个 MCP 工具，另有 20,000 多个合成工具；任务带 Rubric，用户模拟与工具执行环境维护交互，Judge Agent 判断成功，编码任务结合真实沙箱。MCP 只是工具规范来源之一。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#Agent数据合成|原文：基础模型的 Agent 数据]]）

基础报告第 3.2 节的联合强化学习对可验证任务使用结果检查器，对开放任务使用 Self-Critique Rubric Reward，并用任务 Token 预算、PTX Loss 与温度衰减控制成本、遗忘和随机性。Thinking 发布页没有逐项披露这些方法在其后训练中的配置。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#强化学习机制的证据边界|原文：强化学习范围]]）

Thinking 发布页说明后训练对 MoE 组件应用 INT4 Weight-only 量化感知训练，低延迟模式下生成速度约提高 2 倍，并将发布页基准标为 INT4 精度。这是依赖官方服务、硬件与基线的厂商口径。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#原生INT4部署|原文：INT4]]）

## 评测边界

官方发布页报告 HLE 工具设置为 44.9、BrowseComp 为 60.2、SWE-bench Verified 为 71.3。Heavy Mode 并行生成 8 条轨迹再反思汇总，不能直接与单轨迹模式比较；编码任务采用内部 Harness 和 5 次独立运行平均，部分对照分数按相同条件重测。这些结果不构成模型整体能力排名。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#评测应按设置阅读|原文：评测协议]]）

厂商称可连续完成 200～300 次工具调用，但这不是任意任务的可靠性保证。HLE 最多 120 步、每步 48K 推理 Token；Agent Search 最多 300 步、每步 24K。输入超过 256K 时隐藏此前工具输出，长程能力因此依赖工具、预算与上下文管理。（[[raw/sources/模型原理/模型专题/KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent#长程工具调用与测试时扩展|原文：调用与预算]]）

## 关联

- 后训练综合：[[wiki/syntheses/大模型后训练：从模仿到行为选择]]
- 推理表示与执行：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- Token 与长上下文成本：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
