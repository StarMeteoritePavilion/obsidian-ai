---
title: 模型架构：多头注意力与 QKV
source: https://www.bilibili.com/video/BV1of6cBAEuJ
author: 隔壁的程序员老王
published: 2026-02-05
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型架构
  - 注意力机制
  - Transformer
  - 资料摘要
---

# 模型架构：多头注意力与 QKV

原始资料：[[raw/sources/模型原理/模型架构/多头注意力 MultiHeadAttention 是什么|多头注意力 MultiHeadAttention 是什么]]

## 核心结论

多头注意力让同一组词元表示经过多组独立的 Query、Key 和 Value 投影，在不同 Attention Head 中计算上下文关系，再拼接各头结果。它不是为每个头预先指定“语法”或“语义”，而是通过训练形成不同投影；资料中的“语法表”“需求表”和“内容矩阵”只是解释计算目的的教学类比。

## 计算链路

Embedding 为每个词元提供初始向量。Query 表示当前位置要寻找的特征，Key 表示各位置能够提供的特征，二者通过 $QK^T$ 形成关联分数。自回归模型再用因果 Mask 将未来位置设为负无穷，随后按 $\sqrt{d_k}$ 缩放并执行 Softmax，得到每行和为 1 的注意力权重。权重对 Value 加权求和，形成包含上下文关系的新表示：

$$
\operatorname{Attention}(Q,K,V)=\operatorname{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

多头结构并行重复该过程，每个头使用不同投影矩阵，最后按词元拼接输出。Transformer 后续层继续接收这些混合表示。

## 示例与边界

资料以“我用毒毒毒蛇”说明相同字形会因上下文承担名词、动词和修饰作用，并用 $(20,5,100)$ 与 $(60,70,80)$ 的点积得到 9550，展示 Query 与 Key 怎样计算匹配分数。演示中的标签、整数和可读关注点并不是对真实模型隐向量的逐维解释。

资料称早期 GPT-2 的表示可达 1600 维、DeepSeek 为 7168 维，并推测流行商业模型可能超过一万维；这些数字与推测只保留来源口径，不用于推断其他模型。对 BERT 式双向关联与 GPT、Gemini 式因果 Mask 的对比，也只说明两类注意力可见范围，不代表所有模型架构细节完全相同。

## 关联

- 长上下文中的 GQA、DSA 与 MSA：[[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]]
- 稀疏块注意力：[[wiki/sources/模型架构：MoBA 混合块注意力]]
- Attention 之后的稀疏 FFN：[[wiki/sources/模型架构：MoE 稀疏专家路由]]
- Attention 用于层间聚合：[[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]]
- Token 与隐藏表示：[[wiki/sources/模型原理：Token Space 与 Latent Space]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- KV Cache 的推理成本：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
