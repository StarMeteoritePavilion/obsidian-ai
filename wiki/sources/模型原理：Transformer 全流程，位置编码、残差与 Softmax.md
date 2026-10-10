---
title: 模型原理：Transformer 全流程，位置编码、残差与 Softmax
source: https://www.bilibili.com/video/BV15z4C6SEHT?p=6
author: 飞天闪客
published: 2026-08-29
ingested: 2026-10-10
updated: 2026-10-10
tags:
  - AI
  - Transformer
  - 位置编码
  - 残差连接
  - Softmax
  - 模型原理
  - 资料摘要
---

# 模型原理：Transformer 全流程，位置编码、残差与 Softmax

原始资料：[[raw/sources/模型原理/基础原理/Transformer 全流程：位置编码、残差与 Softmax]]

## 核心结论

本讲将简化结构对应到 Transformer 图：词嵌入加入位置编码，重复执行 Attention 与 FFN，每段配残差及归一化，最终 Linear 输出 logits，再由 Softmax 形成词表概率。

位置向量以可训练嵌入说明；残差写为 $y=g(wx+b)+x$，为深层训练提供直通路径。归一化例子的目标是均值接近 0、方差接近 1，不应把该教学形式当成所有模型统一布局。

## 必须分开的数字

回顾加权系数为 $0.2,0.3,0.5$，与另一讲的 $0.1,0.2,0.7$ 分属两个图示。归一化图从 $13,22,6,5,81,4,55,90$ 变为 $-0.6,-0.4,-0.8,-0.9,1.4,-0.9,0.6,1.7$，是教学近似，不是带完整参数的层计算。

Softmax 为 $p_i=e^{x_i}/\sum_j e^{x_j}$；logits $4,2,1,5,3$ 对应「求」「三连」「谢谢」「好」「帅」，图示概率 $0.23,0.03,0.01,0.64,0.09$ 加总为 1。旁白称总和为零，与画面冲突；有效概率分布应归一化为 1。

## 与其他资料的边界

输出 Softmax 把 logits 变为词表概率，注意力内部 Softmax 则归一化关联分数，作用位置不同。这里的可训练位置编码与 [[wiki/sources/模型架构：正弦位置编码与注意力的顺序缺口|正弦编码]]、[[wiki/sources/模型架构：RoPE 相对位置与旋转点积|RoPE]]分别讨论。

本讲没有展开注意力内部缩放和掩码，仅称词表有「5 万多个」项，也没有提供 Dropout 配置。

## 关联

- [[wiki/sources/模型原理：注意力，从加权求和到多头]]：上一集完成加权求和，本集补结构图上剩下的组件。
- [[wiki/sources/模型原理：模型训练，预训练、后训练与思维链]]：下一集讲这些参数怎样从文本里训练出来。
- [[wiki/sources/模型原理：前馈神经网络与线性投影]]
- [[wiki/sources/模型原理：定义问题，预测下一个词]]
- [[wiki/sources/模型架构：简单而强大的 Transformer]]
- [[wiki/sources/模型架构：正弦位置编码与注意力的顺序缺口]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
