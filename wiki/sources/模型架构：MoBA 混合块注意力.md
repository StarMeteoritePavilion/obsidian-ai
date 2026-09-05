---
title: 模型架构：MoBA 混合块注意力
source: https://www.bilibili.com/video/BV1gkPieREDw
author: 唐国梁Tommy
published: 2025-02-25
ingested: 2026-09-03
updated: 2026-09-04
tags:
  - AI
  - 模型架构
  - 注意力机制
  - MoBA
  - 资料摘要
---

# 模型架构：MoBA 混合块注意力

原始资料：[[raw/sources/模型原理/模型架构/月之暗面发布面向大模型的MoBA（混合块注意力）架构 结合MoE和稀疏注意力 算法原理详解|月之暗面 MoBA 算法原理详解]]

## 核心机制

MoBA（Mixture of Block Attention）把 Mixture of Experts 的动态路由思想引入注意力层。系统把上下文划分为 KV 块，为每个 Query 计算块级相关性，通过 Top-k 选择少数历史块，再与当前块分别计算注意力。

当前块启用因果掩码，历史块可以完整读取；路由本身也不能访问未来块。实现先建立 Query 与 KV 块的映射，再重排 Query、使用可变长度 FlashAttention 计算、恢复顺序，并通过 online Softmax 合并当前块与历史块的输出。

## 取舍边界

标准注意力允许每个 Token 与完整历史交互，来源将其计算复杂度记为 $O(N^2)$。MoBA 只激活少量相关块，来源将其复杂度概括为 $O(N\log N)$，并强调可以在全注意力与稀疏注意力之间切换。

稀疏化的收益取决于块大小、Top-k、任务所需的跨块依赖和底层实现。块级路由先使用聚合表示筛选，因此仍需验证关键细节是否因粗粒度选择而丢失。

## 实验边界

资料展示的 Llama-8B-1M 实验中，MoBA 与全注意力在多项短任务及 LongBench、RULER 上整体接近。RULER 128K 分别为 0.7818 与 0.7849，LongBench 32K 分别为 0.4828 与 0.4821；一百万 Token 的 Needle-in-a-Haystack 图基本保持高召回。在该实验配置下，MoBA 相对基于 FlashAttention 的全注意力减少约 6.5 倍计算时间。

这些结果受模型、硬件、块划分、Top-k 和任务设置约束，不能作为其他系统的性能保证。

## 待验证方向

资料明确列出三项后续问题：优化块大小与 Top-k，使稀疏程度适配不同任务；扩展到图像、文本等多模态输入；检查复杂推理中的泛化，确认块级选择是否遗漏关键跨块依赖。这些内容是研究方向，不是现有实验已经证明的结论。

## 与其他长序列方法的关系

MoBA 保留 Transformer，通过块级稀疏路由减少参与注意力的 KV；Memory Caching 面向线性 RNN，通过保存分段隐状态检查点增加可检索历史。两者都在效率与长程召回之间增加可调参数，但修改的是不同架构层。

应用层上下文工程决定哪些外部信息进入窗口，MoBA 则决定窗口内 Token 如何计算注意力，二者不能互相替代。

## 关联

- GQA 与 Token／块级稀疏筛选：[[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]]
- 线性 RNN 的可增长记忆：[[wiki/sources/长序列建模：Memory Caching]]
- 窗口、Attention 与 KV Cache：[[wiki/sources/上下文工程：第二期 窗口与 Token]]
- 应用层上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
