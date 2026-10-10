---
title: 模型架构：简单而强大的 Transformer
source: https://www.bilibili.com/video/BV1NCgVzoEG9?p=6
author: 飞天闪客
published: 2025-07-21
ingested: 2026-10-10
updated: 2026-10-10
tags:
  - AI
  - Transformer
  - 注意力
  - 多头注意力
  - 模型原理
  - 资料摘要
---

# 模型架构：简单而强大的 Transformer

原始资料：[[raw/sources/模型原理/基础原理/简单而强大的 Transformer]]

## 核心结论

Transformer 通过位置编码加入顺序信息，再用可训练 Q/K/V 投影读取上下文。标准缩放点积注意力为：

$$
\mathrm{Attention}(Q,K,V)=\mathrm{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V.
$$

多头分别投影和聚合，拼接后仍有输出映射：

$$
\mathrm{MultiHead}(Q,K,V)=\mathrm{Concat}(\mathrm{head}_1,\ldots,\mathrm{head}_h)W^O,
$$

其中 $\mathrm{head}_i=\mathrm{Attention}(QW_i^Q,KW_i^K,VW_i^V)$。残差与归一化用于深层信息、梯度与数值处理；本讲没有展开其推导。

Encoder 处理完整源文本；Decoder 用因果掩码限制目标侧可见性，并以自身 Q 查询 Encoder 的 K/V。输出投影与 Softmax 形成下一词分布，来源采用取最大概率的教学解码。

## 画面里的数字和式子

翻译例子为“I love you baby”到「我爱你宝贝」，假设词向量 6 维、两个头；「爱」0.7、「喜」0.2 是教学分布中的部分项，不是真实模型输出。结构重复次数只标 $N\times$，未给 $N$。

## 与其他资料的边界

本例没有温度、Top-K 或具体位置编码算法。Decoder-only 的生成路线与完整 Encoder–Decoder 翻译结构分别理解，来源的“GPT 只保留解码器”是宏观概括，不包含具体 GPT 模型规格。

## 关联

- [[wiki/sources/模型架构：从词嵌入到 RNN]]：上一篇。
- [[wiki/sources/模型架构：多头注意力与 QKV]]
- [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- [[wiki/sources/模型架构：正弦位置编码与注意力的顺序缺口]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
