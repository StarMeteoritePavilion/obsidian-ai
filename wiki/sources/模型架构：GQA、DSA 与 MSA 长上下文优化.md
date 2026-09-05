---
title: 模型架构：GQA、DSA 与 MSA 长上下文优化
source: https://www.bilibili.com/video/BV1BY4o6cEXE
author: 隔壁的程序员老王
published: 2026-09-03
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型原理
  - 模型架构
  - 长上下文
  - Sparse-Attention
  - GQA
  - KV-Cache
  - 资料摘要
---

# 模型架构：GQA、DSA 与 MSA 长上下文优化

原始资料：[[raw/sources/模型原理/模型架构/1M上下文的秘密 Sparse Attention, GQA, KV Cache|1M上下文的秘密 Sparse Attention, GQA, KV Cache]]

## 核心结论

长上下文同时增加单步解码的注意力计算和 KV Cache 显存。GQA 让多组 Query Head 共享较少的 KV Head，主要压缩缓存；Sparse Attention 先筛选高贡献 Token 或块，再让主注意力精算，主要控制计算量。两者解决不同成本，可以组合使用。

## 从多头注意力到 GQA

自回归生成会复用已有 Token 的 K 和 V，因此 KV Cache 用显存换取重复计算。传统多头注意力让每个头保存独立 KV；GQA 保留多个 Query Head，却让同组 Query Head 共享 KV。

资料用 64 个注意力头、4 个 KV 组说明这一差异：每组 16 个 Query Head 共享一份 KV，缓存份数由 64 降至 4，即原来的 1/16。这个比例只属于该教学配置，不代表所有 GQA 模型的固定压缩率或质量损失。

## DSA 与 MSA 的筛选粒度

资料以 GPT-2 第一层 Attention 的 Softmax 分布说明，大量位置的权重接近 0，却仍参与全量计算。Sparse Attention 在主注意力前增加较小的打分层，只让得分最高的 Top-N 项进入主注意力。

DeepSeek Sparse Attention（DSA）按 Token 选择；MiniMax Sparse Attention（MSA）先按顺序以 128 Token 分块，使用块内最高分作为块分数，再选择 Top-N 块。选择数可以固定，因此主注意力处理的规模不必随总上下文同比增长。

DeepSeek V4 的 CSA 在 DSA 前增加序列维度压缩：先以四倍压缩率形成压缩 KV，再从其中选择 Top-k；HCA 则以 128 倍压缩率形成较短 KV 并执行 Dense Attention。它们与 GQA 的 Head 共享、DSA 的原始序列筛选和 MSA 的分块评分不是同一机制。（[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性|DeepSeek V4]]）

## 没有消失的成本

打分注意力仍需处理完整上下文，计算与显存仍随长度增加。资料认为它只负责重要性估计，规模通常比主注意力小几十倍，因此当前成本仍可控；它还可以采用 GQA 继续压缩 KV。这个数量级是来源的架构概括，不构成其他模型的固定资源比例。

本资料没有展开开场提到的 Kimi MoBA。现有 MoBA 资料显示，它先对 KV 块做均值池化，再由当前 Query 动态选择 Top-k 历史块；MSA 则按本资料的口径使用块内最高 Token 分数给块评分。二者都进行块级稀疏选择，但块表示、路由和完整计算流程不能视为相同。

## 与上下文工程的边界

GQA、DSA、MSA 和 MoBA 改变模型在既定窗口内怎样缓存和计算；RAG、摘要、修剪与上下文路由决定哪些信息进入窗口。标称窗口扩大不能替代应用层信息治理，应用层压缩也不能消除模型内部的 KV Cache 和注意力成本。

## 定位与关联

官方标题、开场和结尾均无期数标识，内容独立解释 KV Cache、GQA 与 Sparse Attention，因此定位为模型原理／模型架构下的长上下文注意力独立专题。官方“AI技术”合集中的第 20 个位置只记录发布编排，不作为文章期数。

- 多头注意力基础：[[wiki/sources/模型架构：多头注意力与 QKV]]
- 另一种块级稀疏注意力：[[wiki/sources/模型架构：MoBA 混合块注意力]]
- 压缩后稀疏选择：[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性]]
- KV Cache 与推理成本：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- 窗口与 Token：[[wiki/sources/上下文工程：第二期 窗口与 Token]]
- 上下文工程综合：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
