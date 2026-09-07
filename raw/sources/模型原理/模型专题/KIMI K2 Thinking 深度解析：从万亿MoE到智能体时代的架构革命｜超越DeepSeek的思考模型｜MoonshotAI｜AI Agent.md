---
title: Kimi K2 Thinking：MoE 预训练、Agent 后训练与 INT4 部署
source: https://www.bilibili.com/video/BV1sJCnBGESj
author: 唐国梁Tommy
created: 2025-11-12
tags:
  - AI
  - 模型原理
  - 大语言模型
  - Kimi-K2-Thinking
  - MoE
  - Agent
updated: 2026-09-07
---

# Kimi K2 Thinking：MoE 预训练、Agent 后训练与 INT4 部署

Kimi K2 Thinking 是 Moonshot AI 在 Kimi K2 基础上发布的推理与工具调用模型。理解它需要分清两组资料：Kimi K2 技术报告解释基础模型的预训练、数据与 Agent 后训练；Kimi K2 Thinking 模型卡和发布页说明 Thinking 版本的架构摘要、INT4 部署、长程工具调用与评测。本文不把基础模型报告中的全部训练方法自动归为 Thinking 版本新增设计。

## 架构摘要

官方模型卡给出的 Kimi K2 Thinking 配置如下：

| 项目 | 官方模型卡数值 |
| --- | ---: |
| 总参数 | 1T |
| 每个 Token 激活参数 | 32B |
| 层数 | 61，包含1个Dense层 |
| Attention Hidden Dimension | 7168 |
| 注意力头 | 64 |
| 专家数 | 384 |
| 每个Token选择的专家数 | 8 |
| 共享专家数 | 1 |
| 词表 | 160K |
| 上下文 | 256K |
| 注意力机制 | MLA |
| 激活函数 | SwiGLU |

Kimi K2技术报告正文把基础模型写为1.04T总参数，模型卡用1T作摘要，两种写法属于精确值与取整值的差异。32B激活参数表示一次前向只使用稀疏专家中的一部分，不能据此直接推出端到端速度或显存按相同比例下降；权重加载、专家通信、KV Cache和内核实现仍会影响部署成本。

## 基础模型如何稳定完成预训练

Kimi K2技术报告第2.1节介绍MuonClip。它把Muon、权重衰减、更新量RMS匹配和QK-Clip组合起来。QK-Clip读取前向计算中已经得到的各注意力头最大logit；当某头超过阈值时，缩放产生该头logit的相关Query和Key权重。报告在完整训练中使用阈值 $\tau=100$，并称Kimi K2在15.5T Token预训练中没有出现Loss Spike。

这里的结论属于Kimi K2基础模型训练记录。它说明Thinking版本所继承的基座怎样训练，不证明任何使用MuonClip的模型都能避免训练不稳定。

## Data Rephrasing提高高质量数据利用率

技术报告第2.2节把知识数据改写拆成三步：多风格和多视角提示、长文档分块自回归改写、原文与改写结果的一致性检查。报告用早期K2检查点比较三种训练设置：

| 改写次数 | Epoch | SimpleQA准确率 |
| ---: | ---: | ---: |
| 0，原始Wiki文本 | 10 | 23.76 |
| 1 | 10 | 27.39 |
| 10 | 1 | 28.94 |

这些数字来自技术报告Table 1，只证明该检查点、数据和训练设置下的结果。报告同时说明，大规模知识语料实际最多改写两次，因此不能把“改写10次”当成生产配方。

## Agent数据合成

技术报告第3.1.1节给出三阶段数据流：先建立真实与合成工具规范库，再为抽样工具组合生成Agent和带成功标准的任务，最后生成并过滤多轮工具调用轨迹。

真实工具部分来自GitHub上的3000多个MCP工具；合成部分通过领域层级扩展生成20000多个工具。每项任务带明确Rubric；User Simulation生成多轮请求，Tool Execution Environment维护工具执行后的状态，Judge Agent按Rubric判断轨迹是否成功。编码等需要真实执行反馈的任务会把模拟器与真实沙箱结合。MCP是工具规范来源之一，不能概括为全部Agent训练数据。

## 强化学习机制的证据边界

Kimi K2技术报告第3.2节描述的是K2基础模型的联合强化学习：可验证任务使用结果检查器，开放式任务使用Self-Critique Rubric Reward。训练还通过按任务设置最大Token预算、辅助PTX Loss和温度衰减，分别控制输出成本、缓解遗忘并在训练后期减少随机性。

Kimi K2 Thinking官方发布页没有逐项声明这些基础模型方法在Thinking后训练中的配置。因此，这些内容只用于解释模型家族已有的训练基础；不能把它们写成Thinking版本独有的完整训练配方。

## 原生INT4部署

Thinking发布页说明，后训练阶段对MoE组件应用INT4 Weight-only量化感知训练。官方称低延迟模式下生成速度约提高2倍，并将发布页中的全部基准结果标为INT4精度。这个数字是官方发布口径，依赖其推理服务、硬件和基线，不能外推到任意本地量化方案。

## 长程工具调用与测试时扩展

官方把Kimi K2 Thinking描述为能交错执行推理和函数调用，并称其可以连续完成200至300次工具调用。该说法来自模型卡和发布页的厂商测试，不是任意任务上的可靠性保证。

发布页的Agent评测设定更具体：HLE配备搜索、代码解释器和网页浏览工具，最大120步，每步48K推理Token预算；Agent Search任务最大300步，每步24K推理Token预算。输入超过256K时，评测会隐藏之前的工具输出。这些条件说明“长程”能力依赖工具、步数预算和上下文管理，不能只看调用次数。

## 评测应按设置阅读

官方发布页报告Kimi K2 Thinking在HLE工具设置为44.9、BrowseComp为60.2、SWE-bench Verified为71.3。Heavy Mode先并行生成8条轨迹，再反思汇总最终结果，因此Heavy分数不能与单轨迹默认模式直接比较。编码任务使用官方内部评测Harness，结果为5次独立运行的平均值；部分对照分数由官方在相同条件下重测并用星号标出。

这些结果用于描述特定版本在公开基准与官方Harness中的表现，不足以证明它在所有搜索、编程或工具调用任务上优于其他模型。

## 来源与版本

| 来源 | 版本与定位 | 支持范围 |
| --- | --- | --- |
| [Kimi K2 Thinking官方模型卡](https://huggingface.co/moonshotai/Kimi-K2-Thinking) | 2026-09-07核查；“Key Features”“Model Summary” | 架构摘要、INT4、200至300次工具调用 |
| [Kimi K2 Thinking官方发布页](https://moonshotai.github.io/Kimi-K2/thinking.html) | 2026-09-07核查；“Inference Efficiency”“Full Evaluations”及脚注 | INT4范围、评测数字、工具和预算设置、Heavy Mode |
| [Kimi K2技术报告](https://arxiv.org/html/2507.20534v2) | arXiv:2507.20534v2，2026-02-03；第2.1、2.2、2.3、3.1.1、3.2节 | 基础模型MuonClip、数据改写、架构、Agent数据合成与RL；不冒充Thinking版本新增方法 |
| [原视频](https://www.bilibili.com/video/BV1sJCnBGESj) | 唐国梁Tommy，2025-11-12 | 文章主题和讲解线索；关键数字以以上官方资料为准 |
