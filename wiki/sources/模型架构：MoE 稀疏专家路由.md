---
title: 模型架构：MoE 稀疏专家路由
source: https://www.bilibili.com/video/BV1CgZABxEcy
author: 隔壁的程序员老王
published: 2026-02-19
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型架构
  - MoE
  - 混合专家
  - DeepSeek
  - 资料摘要
---

# 模型架构：MoE 稀疏专家路由

原始资料：[[raw/sources/模型原理/模型架构/MoE为什么这么快 —— 从小学数学到MoE 大模型进化史|MoE为什么这么快 —— 从小学数学到MoE 大模型进化史]]

## 核心结论

MoE 用多个较小 FFN 替换一个大型稠密 FFN，再由 Router 为每个 Token 选择少量路由专家并加权汇总。总参数决定模型可容纳的专家容量，激活参数更直接决定单次计算量；稀疏激活让模型扩大总容量，而不必让全部参数参与每个 Token 的计算。

## 从 FFN 到 MoE

线性层与非线性激活组成 FFN。单独按位置执行的 FFN 无法区分“老王爱”与“恋爱”中的“爱”；Attention 先形成带有上下文的表示，再交给 FFN。Dense 模型用同一组庞大 FFN 参数处理所有 Token，MoE 则把 FFN 拆成多个 Expert，并加入 Router 动态选择。

资料以 DeepSeek-V3 为例：每层有 256 个路由专家，每个 Token 选择 8 个。被称作“数学专家”或“情感专家”的功能只是教学类比，真实专家分工在训练中形成，不能由人类直接命名。DeepSeek 的共享专家不经过 Router，每次都参与计算，用于承载通用知识。

## 对比数据与边界

DeepSeekMoE 论文表格中，DeepSeek 67B Dense 的总参数和激活参数均为 67.4B，每 4K Token FLOPs 为 2057.5T，Pile Loss 为 1.905；DeepSeekMoE 145B 的总参数为 144.6B、激活参数为 22.2B、每 4K Token FLOPs 为 585.6T、Pile Loss 为 1.876。

资料将其概括为“约两倍空间换约三倍性能”。该结论来自对应训练设置；表格记录的是总参数、激活参数、FLOPs 与 Loss，不直接等于所有硬件上的端到端速度、Token 成本或服务价格。

DeepSeek V4 进一步处理 MoE 的训练稳定性：Anticipatory Routing 使用历史路由参数预先计算专家索引，打断路由概率与专家参数更新之间的即时正反馈；SwiGLU Clamping 同时限制异常激活。该机制解决的是长时间训练中的 Loss Spike，不改变 MoE 通过稀疏激活分离总容量与单 Token 计算量的基本结构。（[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性|DeepSeek V4]]）

## 关联

- Linear、Activation 与 MLP：[[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- 上下文形成机制：[[wiki/sources/模型架构：多头注意力与 QKV]]
- 块级稀疏注意力：[[wiki/sources/模型架构：MoBA 混合块注意力]]
- Kimi K2 Thinking 的 MoE 实例：[[wiki/sources/大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练]]
- Qwen 3.5 的 397B／17B 与 10 路由专家＋1 共享专家实例：[[wiki/sources/大语言模型：Qwen 3.5 的 MoE、混合注意力与应用演示]]
- DeepSeek V4 的路由稳定性：[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性]]
- 模型推理与成本：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
