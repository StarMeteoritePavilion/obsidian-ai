---
title: 模型架构：MoE 路由、Top-K 与负载均衡
source: https://www.bilibili.com/video/BV1HW4X6QEuh
author: 张司机在路上
published: 2026-08-30
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - MoE
  - 混合专家
  - Router
  - Top-K
  - 负载均衡
  - 模型架构
  - 模型原理
  - 资料摘要
---

# 模型架构：MoE 路由、Top-K 与负载均衡

原始资料：[[raw/sources/模型原理/模型架构/MoE混合专家架构如何让模型参数越做越大|MoE混合专家架构如何让模型参数越做越大]]

## 核心结论

MoE 把稠密 FFN 扩展为多个 Expert，由 Router 为每个 Token 选择少量路径。总专家数可以增加模型容量，而每个 Token 的活跃专家数不必同比增加；模型总参数量与单 Token 活跃计算量因此需要分开统计。

完整前向过程是“路由打分—Dispatch—Expert 计算与概率加权—Combine”。Top-K 路由让多个高分专家分别处理同一 Token，再对选中分数归一化并混合结果。资料的发布时示例称 DeepSeek-V4 从 384 个专家中选择 6 个，Kimi-K3 从 896 个专家中选择 16 个；这些数字只属于对应模型示例。

## Router 与梯度

资料将 Router 表示为小型 FFN 后接 Softmax。Top-1 选择中的 `argmax` 本身不可导，但选中概率会乘入 Expert 输出，使梯度能够回传到 Router 的评分参数。另一份资料将 Router 教学性地简化为普通线性层；两份来源的结构口径不同，当前资料没有给出具体模型实现，不能把任一表述写成所有 MoE 的固定 Router 结构。（[[wiki/sources/模型架构：MoE 稀疏专家路由|另一份 MoE 资料]]）

## 负载均衡

随机初始化时，某个 Expert 若先收到更多 Token，就会得到更多梯度并进一步吸引路由，形成专家坍塌。资料用 $P_i$ 表示 Router 对 Expert $i$ 的平均概率，用 $f_i$ 表示该 Expert 的实际选择频率，并定义：

$$
\operatorname{Auxiliary\ Loss}=\alpha E\sum_{i=1}^{E}f_iP_i
$$

画面中的集中路由示例为 `P = [0.58, 0.19, 0.11, 0.12]`、`f = [1, 0, 0, 0]`，暂不计 $\alpha$ 时损失显示为 `2.33`；均衡示例为 `P = [0.28, 0.23, 0.27, 0.22]`、`f = [0.25, 0.25, 0.25, 0.25]`，损失为 `1.00`。辅助损失推动负载趋于均衡，但资料没有给出真实模型的训练曲线、系数设置或效果消融。

## 证据边界

- “增加专家而训练、推理时间几乎不变”是资料基于固定活跃专家数给出的结构性概括，没有端到端计时、通信、存储或硬件对照，不能写成所有 MoE 部署的固定性能结论。
- 标点、动词、连词和数字专家是教学标签，不能据此认定真实 Expert 具有可直接命名的固定分工。
- DeepSeek-V4 与 Kimi-K3 的专家总数和 Top-K 来自视频画面及旁白，属于视频发布时记录，不与其他模型或版本混用。

## 关联

- [[wiki/sources/模型架构：MoE 稀疏专家路由]]
- [[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性]]
- [[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- [[wiki/sources/模型架构：多头注意力与 QKV]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- [[wiki/syntheses/深层模型训练稳定性：残差、更新与路由]]
