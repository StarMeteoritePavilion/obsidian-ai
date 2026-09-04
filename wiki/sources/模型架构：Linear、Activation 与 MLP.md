---
title: 模型架构：Linear、Activation 与 MLP
source: https://www.bilibili.com/video/BV1i5koBtEUU
author: 隔壁的程序员老王
published: 2025-11-13
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型架构
  - Transformer
  - Linear
  - MLP
  - 资料摘要
---

# 模型架构：Linear、Activation 与 MLP

原始资料：[[raw/sources/模型原理/模型架构/从Linear到MLP AI模型的数学本质【Transformer结构拆解】|从Linear到MLP AI模型的数学本质【Transformer结构拆解】]]

## 核心结论

Linear 用可训练的权重和偏置完成向量间的线性映射；多个 Linear 直接串联后仍然只能表示线性关系。在线性层之间加入 ReLU、Sigmoid、tanh 或 GELU 等非线性激活函数，才能组成可以拟合复杂函数的 FFN／MLP。Transformer 的 Feed Forward 模块建立在这类结构之上。

## 线性映射与训练

资料以活动数量到热量和耗时的映射说明 $y=Wx+b$：与输入相乘的参数是 Weight，额外加入的常量是 Bias。GPT-2 XL 输出端的 Linear 则把 1600 维内部表示映射为 50257 个 Token 的匹配分数。

手写参数可以逐项解释，大模型参数则由训练得到。梯度下降根据预测结果与正确结果的差距反复调整参数；训练完成只说明整组参数能够实现相应映射，不代表每个数字都具有稳定、可读的单独含义。

## 非线性与示例模型

两个线性函数复合后仍然是线性函数，因此无法拟合先上升、再趋平、最后下降的曲线。ReLU 以 $\max(0,x)$ 引入非线性拐点；Sigmoid 与 tanh 更平滑，但资料指出它们在输入绝对值较大时变化很小，容易造成梯度消失；GELU 则没有完全舍弃负数区域。

示例 MLP 使用 `1→128→256→1` 三个 Linear，前两层后接 ReLU，共有 33537 个可训练参数，以 2000 条随机训练数据拟合三段“吃货函数”。这些数字属于视频演示，不构成其他网络的默认结构或训练规模。

## 系列定位与关联

官方“从0开始一起学大模型”合集将本文列为第二条，上一条为《Transformer如何成为AI模型的地基》。视频明确承接其宏观架构介绍，但没有使用“第二期”标识，因此本库记录合集顺序，不另行命名期数。

- 系列导论：[[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- Attention 与上下文表示：[[wiki/sources/模型架构：多头注意力与 QKV]]
- FFN 的稀疏专家化：[[wiki/sources/模型架构：MoE 稀疏专家路由]]
- 残差通道与层间聚合：[[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]]
- 模型内部表示：[[wiki/sources/模型原理：Token Space 与 Latent Space]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
