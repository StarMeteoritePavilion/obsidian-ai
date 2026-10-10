---
title: 模型架构：多头注意力与 QKV
source: https://www.bilibili.com/video/BV1of6cBAEuJ
author: 隔壁的程序员老王
published: 2026-02-05
ingested: 2026-09-04
updated: 2026-10-10
tags:
  - AI
  - 模型架构
  - 注意力机制
  - Transformer
  - 资料摘要
---

# 模型架构：多头注意力与 QKV

原始资料：[[raw/sources/模型原理/模型架构/多头注意力 MultiHeadAttention 是什么|多头注意力：从 Q、K、V 到上下文表示]]

## 核心结论

多头注意力对当前子层表示执行多组独立 Q、K、V 投影，各头读取可见上下文，再拼接并经过输出投影 $W^O$。头的功能由训练形成；“语法表”“需求表”等名称是教学类比，不是预先分配的真实功能。

## 计算链路

标准单头计算为：

$$
H_i=\operatorname{softmax}_{\text{Key轴}}\left(\frac{Q_iK_i^T}{\sqrt{d_k}}+M\right)V_i.
$$

Softmax 对每个 Query、每个头沿 Key 轴归一化。因果 Mask 屏蔽未来，Padding Mask 屏蔽补齐；整行不能全部被屏蔽而造成无效归一化。$\sqrt{d_k}$ 缩放用于避免点积随维度增长而使 Softmax 过度饱和，其动机依赖论文给定的随机变量假设。（[[raw/sources/模型原理/模型架构/多头注意力 MultiHeadAttention 是什么#从分数到上下文|缩放与掩码]]）

标准配置 $d_k=d_v=d/h$ 下，输入为 `[B,T,d]`，分头后为 `[B,h,T,d_k]`，分数为 `[B,h,T,T]`，聚合结果拼接回 `[B,T,h*d_v]`，再投影到 `[B,T,d]`。只有第一层入口直接接收词嵌入和位置信息，后续层使用前一层表示。（[[raw/sources/模型原理/模型架构/多头注意力 MultiHeadAttention 是什么#投影、分头与形状|形状]]、[[raw/sources/模型原理/模型架构/多头注意力 MultiHeadAttention 是什么#拼接之后还有输出投影|输出投影]]）

## 示例与边界

$(20,5,100)\cdot(60,70,80)=9550$ 是纯算术教学示例，不代表真实隐向量的逐维语义或数值范围。

Encoder 自注意力从同一编码层输入取得 Q/K/V，读取全部有效源位置；Decoder 因果自注意力读取当前及此前位置；交叉注意力的 Q 来自 Decoder，K/V 来自 Encoder，Query 与 Key 长度可以不同。（[[raw/sources/模型原理/模型架构/多头注意力 MultiHeadAttention 是什么#三种注意力的输入来源|输入与可见范围]]）

## 关联

- Attention 的顺序缺口与正弦位置编码：[[wiki/sources/模型架构：正弦位置编码与注意力的顺序缺口]]
- Query／Key 的相对位置旋转：[[wiki/sources/模型架构：RoPE 相对位置与旋转点积]]
- 长上下文中的 GQA、DSA 与 MSA：[[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]]
- 稀疏块注意力：[[wiki/sources/模型架构：MoBA 混合块注意力]]
- Attention 之后的稀疏 FFN：[[wiki/sources/模型架构：MoE 稀疏专家路由]]
- Attention 用于层间聚合：[[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]]
- Token 与隐藏表示：[[wiki/sources/大语言模型：Token、Embedding 与 Latent Space]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- KV Cache 的推理成本：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
