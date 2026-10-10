---
title: 模型架构：GQA、DSA 与 MSA 长上下文优化
source: https://www.bilibili.com/video/BV1BY4o6cEXE
author: 隔壁的程序员老王
published: 2026-09-03
ingested: 2026-09-04
updated: 2026-10-10
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

原始资料：[[raw/sources/模型原理/模型架构/1M上下文的秘密 Sparse Attention, GQA, KV Cache|百万 Token 上下文的成本：KV Cache、GQA 与稀疏注意力]]

## 核心结论

密集注意力的 Prefill 含 $O(N^2d)$ 计算；使用 KV Cache 的 Decode 每步扫描历史约为 $O(Nd)$。GQA 主要减少缓存份数，稀疏注意力减少主注意力实际读取的位置，二者可组合但不能用同一比例描述。（[[raw/sources/模型原理/模型架构/1M上下文的秘密 Sparse Attention, GQA, KV Cache#Prefill与Decode的计算不同|计算阶段]]）

## 从多头注意力到 GQA

MHA 为每个 Query Head 保存独立 KV；MQA 全部共享一组；GQA 按组共享。64 个 Query Head、4 个 KV Head 的教学配置，在层数、长度、头维度和精度不变时，缓存为对应 MHA 的 $1/16$。每个 Query Head 仍执行注意力，计算量不随缓存份数统一降到 $1/16$。（[[raw/sources/模型原理/模型架构/1M上下文的秘密 Sparse Attention, GQA, KV Cache#MHA、MQA与GQA共享的是什么|共享对象]]）

## DSA 与 MSA 的筛选粒度

- **DSA**：DeepSeek-V3.2 v1 第 2.1 节以 Lightning Indexer 为历史 Token 评分，选择细粒度 KV；报告的稀疏训练配置每个 Query 选择 2048 个 KV Token。
- **NSA**：论文 v2 第 3.2–3.3 节结合压缩块、重要块选择和滑动窗口三个分支，不等同于 DSA 的 Token Top-k。
- **MSA**：论文 v2 第 3.1 节按 Query 与 GQA 组选择块，以轻量 Index Branch 的 Token 分数做块内 max-pooling，选 Top-k 并保留当前局部块。第 5 节实验块长 128，主分支只读取选块中因果可见的位置。

缩写、选择单位与缓存共享方式分别以对应原论文为准。（[[raw/sources/模型原理/模型架构/1M上下文的秘密 Sparse Attention, GQA, KV Cache#稀疏选择：减少主注意力实际读取的位置|三种机制及准确版本]]）

## 没有消失的成本

固定精算 Top-k 不会消除 Indexer 扫描、选取、KV 存储与搬运。打分器开销取决于确切结构和精度，没有统一“小几十倍”的比例。MoBA 用块级均值表示路由，MSA 用块内最高 Token 分数，不能把两者当成同一流程。（[[raw/sources/模型原理/模型架构/1M上下文的秘密 Sparse Attention, GQA, KV Cache#打分和路由并不是免费计算|剩余成本]]）

## 与上下文工程的边界

模型架构决定窗口内的缓存和计算；RAG、摘要、修剪与路由决定哪些信息进入窗口。扩大窗口不能替代应用层治理，应用层压缩也不消除内部计算成本。

## 定位与关联

- 多头注意力基础：[[wiki/sources/模型架构：多头注意力与 QKV]]
- 另一种块级稀疏注意力：[[wiki/sources/模型架构：MoBA 混合块注意力]]
- 压缩后稀疏选择：[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性]]
- KV Cache 与推理成本：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- 窗口与 Token：[[wiki/sources/上下文工程：第二期 窗口与 Token]]
- 上下文工程综合：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
