---
title: 模型架构：Attention Residuals 层间选择性聚合
source: https://www.bilibili.com/video/BV1MRX2B2ECg
author: 隔壁的程序员老王
published: 2026-04-02
ingested: 2026-09-04
updated: 2026-10-10
tags:
  - AI
  - 模型架构
  - 残差连接
  - Attention-Residuals
  - Kimi
  - 资料摘要
---

# 模型架构：Attention Residuals 层间选择性聚合

原始资料：[[raw/sources/模型原理/模型架构/注意力残差是什么？ [白话读论文]|Attention Residuals：让深层网络选择前序表示]]

## 核心结论

AttnRes 沿深度轴选择历史子层输出，区别于沿序列轴读取 Token 的注意力。标准残差以固定系数 1 累加输出；Full AttnRes 让每个子层持有可训练 pseudo-query，对输入嵌入及此前子层输出打分，再沿前序层轴做 Softmax 聚合。（[[raw/sources/模型原理/模型架构/注意力残差是什么？ [白话读论文]#Full AttnRes：每层一个pseudo-query|Full 机制]]）

## 从标准残差到 AttnRes

pseudo-query 按子层独立，不是由当前 Token 再投影产生；聚合权重仍随输入变化，因为历史输出随 Token 变化。RMSNorm 只用于打分的 Key，Value 使用原始输出。这里聚合的是前序子层输出，而非各层输入的残差和。

权重构成凸组合，聚合结果的范数不超过所选输出的最大范数；被聚合输出自身仍可增长，因此权重和为 1 不构成整个网络无条件稳定的证明。

## Block AttnRes 与开销边界

Block AttnRes 将同块子层输出相加，保留已完成块的摘要与当前块部分和。每个 Attention 或 MLP 子层前都会查询输入嵌入、历史块和当前部分和；块边界只负责归档与重置。分块减少历史表示数量，同时牺牲对已完成块内各子层的独立选择。（[[raw/sources/模型原理/模型架构/注意力残差是什么？ [白话读论文]#Block AttnRes：保存块摘要，仍在每个子层前聚合|Block 与 Figure 2]]）

按单 Token、固定隐藏维度 $d$ 计，Full 的全深度工作量为 $O(L^2d)$，历史输出为 $O(Ld)$；Block 将历史表示降到 $O(Nd)$，全深度查询约含 $O(LNd)$。普通训练已为反向传播保存相关激活，额外存储主要受激活重算、流水线与通信方式影响，不能统一写成新增固定显存。

论文 v1 在 48B 总参数、3B 激活参数的 Kimi Linear 架构上训练 1.4T Token，报告更均匀的层输出、梯度及所测下游任务改进。库内没有独立复现该训练；结果绑定这组模型与协议。（[[raw/sources/模型原理/模型架构/注意力残差是什么？ [白话读论文]#深度、显存与带宽的代价|开销与实验]]）

mHC 约束多通道残差的混合矩阵，AttnRes 则选择历史表示；两者修改残差的方式不同。（[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性|DeepSeek V4]]）

## 关联

- Linear、Activation 与 FFN：[[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- Token 间注意力：[[wiki/sources/模型架构：多头注意力与 QKV]]
- FFN 与 MoE：[[wiki/sources/模型架构：MoE 稀疏专家路由]]
- Residual Stream 与内部表示：[[wiki/sources/大语言模型：Token、Embedding 与 Latent Space]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
