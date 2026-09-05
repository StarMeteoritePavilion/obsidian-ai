---
title: 模型架构：DeepSeek V4 的长上下文与训练稳定性
source: https://www.bilibili.com/video/BV153oRBXEsG
author: 唐国梁Tommy
published: 2026-04-25
ingested: 2026-09-04
updated: 2026-09-05
tags:
  - 模型专题
  - AI
  - 模型原理
  - 模型架构
  - DeepSeek-V4
  - 长上下文
  - MoE
  - 资料摘要
---

# 模型架构：DeepSeek V4 的长上下文与训练稳定性

原始资料：[[raw/sources/模型原理/模型专题/DeepSeek V4 硬核全面拆解｜1.6万亿参数+百万级上下文+33万亿token同时不崩，CSA、mHC、Muon、OPD四大核心模块逐条拆解|DeepSeek V4 硬核全面拆解]]

## 核心结论

DeepSeek V4 不是依靠单个模块支持百万 Token 上下文与 1.6 万亿参数，而是同时约束序列计算、残差信号、权重更新、MoE 路由和后训练整合。CSA 与 HCA 组合压缩和稀疏选择；mHC 把残差混合限制在 Birkhoff 多面体；Muon 近似正交化二维矩阵更新；Anticipatory Routing 打断路由与专家参数的即时正反馈；OPD 再把多个领域专家合并到统一模型。

## 长上下文的两级压缩

CSA 先以四倍压缩率把百万 Token 转成约 25 万个压缩 KV，再为当前 Query 选择 Top-k；DeepSeek-V4-Pro 使用 1024 项。由于相邻压缩块采用重叠窗口，每个压缩块覆盖约八个原始 Token 的语义，1024 个压缩块等效覆盖约 8000 个原始 Token。HCA 以 128 倍压缩率把序列压到约 8000 个 KV，并直接执行 Dense Attention。两者在层间交错，分别保留较细粒度选择和重压缩后的全局视野。

在百万 Token 上下文下，资料给出的 DeepSeek-V4-Pro 单 Token 推理 FLOPs 和 KV Cache 分别约为 DeepSeek-V3.2 的 27% 与 10%。资料还给出 4.6GB、15GB 与 31GB 的 V4-Pro、V3.2 MLA 和 GQA-8 KV Cache 对照；具体数值依赖模型配置、精度与 Serving 口径。

## 三类训练稳定性约束

mHC 把四通道残差混合矩阵限制为非负双随机矩阵，并用 Sinkhorn-Knopp 交替归一化约 20 轮。Birkhoff 多面体中的矩阵谱范数不超过 1，连续相乘仍保持约束，因此可以限制跨层信号与梯度持续放大。

Muon 用 Hybrid Newton-Schulz 近似正交化大部分二维权重矩阵的更新；前八步快速拉近奇异值，后两步精修。Embedding、预测头、mHC 部分参数和 RMSNorm 权重仍使用 AdamW。它与 mHC 分别约束权重更新和残差传播。

Anticipatory Routing 用历史路由参数预先计算当前步骤的专家索引，同时保留当前参数的特征计算、Gating 和专家前向，以打断路由概率与专家更新的即时正反馈。该机制不是常态路由方式：系统只在检测到 Loss Spike 时短暂回滚并启用，稳定后恢复标准训练。SwiGLU Clamping 进一步限制异常值。技术报告确认两项方法有效，但没有给出完整理论解释。

## OPD 与能力边界

OPD 先独立训练数学、代码、Agent 和指令遵循等专家，再让 Student 沿自身轨迹优化相对 Teacher 的 Reverse KL。新增领域可以独立加入 Teacher 池，不必重做统一 Mixed RL 的奖励调度。它与视觉原语资料中的 OPD 都采用“先专家、后统一”的路径，但专家类型、任务和训练目标不能直接等同。

评测结果并不支持“所有任务全面领先”。Codeforces 评分为 3206，Putnam-2025 在高计算量混合证明流程下达到 120/120；MRCR 1M 为 83.5，低于 Claude Opus 4.6 的 92.9；Apex 为 38.3，低于 Gemini-3.1-Pro 的 60.9。Agent 各项结果差异也不一致。资料估算百万 Token 上下文下每生成一个 Token 的成本约为 0.00089 美元，该数字同样依赖模型模式与服务口径。

复现还依赖 DeepGEMM、TileLang、3FS、DeepSeek Elastic Compute（DSec）和 DualPipe 等基础设施，mHC、Muon、CSA 的归一化与 Attention 数值边界也彼此耦合。任何单点简化都可能改变其他模块的稳定性前提。

## 关联

- 稀疏注意力与 KV Cache：[[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]]
- 残差连接的另一条改造路线：[[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]]
- MoE 与专家路由基础：[[wiki/sources/模型架构：MoE 稀疏专家路由]]
- 另一种 Muon 稳定化实例：[[wiki/sources/大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练]]
- 视觉专家的 OPD 实例：[[wiki/sources/多模态推理：视觉原语的数据、训练与奖励]]
- 后训练机制对照：[[wiki/syntheses/大模型后训练：从模仿到行为选择#专家分训与统一模型：SFT、RL 和 OPD|SFT、RL 与专家统一蒸馏]]
- 推理表示与执行综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
