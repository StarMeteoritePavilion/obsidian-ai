---
title: 模型架构：Attention Residuals 层间选择性聚合
source: https://www.bilibili.com/video/BV1MRX2B2ECg
author: 隔壁的程序员老王
published: 2026-04-02
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型架构
  - 残差连接
  - Attention-Residuals
  - Kimi
  - 资料摘要
---

# 模型架构：Attention Residuals 层间选择性聚合

原始资料：[[raw/sources/模型原理/模型架构/注意力残差是什么？ [白话读论文]|注意力残差是什么？ [白话读论文]]]

## 核心结论

标准残差连接以固定单位权重累加各层输出，随着深度增加可能出现隐藏表示数值膨胀和单层信息被累计总和稀释。Attention Residuals（AttnRes）借用 Attention 的选择性聚合思想，让当前层通过可训练 pseudo-query 为此前表示分配 Softmax 权重，再加权形成当前层输入。

## 从标准残差到 AttnRes

残差连接为每个模块提供跨层直通路径，使训练信号不必只沿完整模块链逐层传播，资料据此将其与缓解梯度消失联系起来。代价是每层输出都被直接加入同一通道：深层接收的总和越来越大，也难以区分此前各层的独立贡献。

Full AttnRes 将固定相加改为对所有此前层输出的动态聚合。Softmax 权重之和为 1，使结果不随层数直接累加；不同层获得不同权重，又让当前模块能够侧重与自身更相关的早期表示。视频以“人物识别层”和“人物关系层”说明这种选择可能形成正向反馈，同时明确这些功能只是教学类比，不能由人类直接指定。

## Block AttnRes 与开销边界

Full AttnRes 需要计算所有跨层权重，并保存和读取此前各层输出，因此增加计算、显存和带宽压力。Block AttnRes 把多个层组成块：块内保留标准残差，块间执行注意力聚合，以较粗粒度换取更低开销。

视频没有介绍论文的实验成绩，因此本摘要不补入外部结果。当前资料支持的是机制与成本取舍，不足以据此判断 AttnRes 在其他模型、规模和硬件上的固定收益。

DeepSeek V4 的 mHC 是另一条残差改造路线：它把残差流扩成多通道，再把混合矩阵约束为 Birkhoff 多面体中的双随机矩阵，以限制深层信号放大。AttnRes 侧重跨层选择，mHC 侧重多通道表达与数值稳定，两者不能因都修改残差路径而视为同一实现。（[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性|DeepSeek V4]]）

## 关联

- Linear、Activation 与 FFN：[[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- Token 间注意力：[[wiki/sources/模型架构：多头注意力与 QKV]]
- FFN 与 MoE：[[wiki/sources/模型架构：MoE 稀疏专家路由]]
- Residual Stream 与内部表示：[[wiki/sources/模型原理：Token Space 与 Latent Space]]
- 多通道约束残差：[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
