---
title: 长序列建模：Memory Caching
source: https://www.bilibili.com/video/BV1GHP4zZESk
author: 唐国梁Tommy
published: 2026-03-06
ingested: 2026-09-02
updated: 2026-10-10
tags:
  - AI
  - 模型架构
  - 长序列建模
  - Memory Caching
  - 资料摘要
---

# 长序列建模：Memory Caching

原始资料：[[raw/sources/模型原理/模型架构/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了|RNN 不再“金鱼记忆”]]

## 核心问题

密集 Transformer 的 KV Cache 容量随历史长度线性增长，训练与 Prefill 含平方级注意力计算，缓存 K/V 后的 Decode 单步扫描为线性计算。固定容量的递归记忆面临长程召回瓶颈；Memory Caching 保存分段状态，使可查询的历史容量随段数增长。Linear Attention、Mamba 与 Titans 的内部结构不同，不能全部归为同一种传统 RNN。（[[raw/sources/模型原理/模型架构/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了#先分清状态容量与计算量|原文：容量与计算量]]）

这一问题位于模型架构层，不等同于应用层的上下文裁剪、RAG 或摘要压缩。二者都处理有限信息预算，但前者改变模型如何保存和查询序列状态，后者决定哪些外部信息进入当前窗口。

## 方法结构

序列按固定长度切段并保存段末状态。当前 Token 查询段内在线记忆与已完成历史段，再由跨段聚合器组合结果。段间既可接续上一段终态，也可独立初始化；历史状态参与训练时，梯度路径依具体实现，不能把推理缓存直接当作梯度截断。（[[raw/sources/模型原理/模型架构/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了#分段写入、缓存与查询|原文：分段与训练边界]]）

资料介绍四种策略：Residual Memory 等权聚合所有段；GRM 根据当前 Token 与各段摘要的相关性动态分配权重；Memory Soup 先组合记忆参数再查询；Sparse Selective Caching（SSC）只加载和查询选中的 Top-k 状态，但对全部摘要评分与选段的成本仍随历史段数增长，不能把完整检索成本写成固定值。（[[raw/sources/模型原理/模型架构/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了#四种聚合策略|原文：聚合策略]]）

当前原文已按论文 v1 第 3.1 节将 GRM 明确为 Gated Residual Memory。GRM 与 Memory Soup 仅在查询算子对记忆参数线性且使用相同权重时等价；DLA、Titans 的深度或非线性记忆不能自动满足该条件。（[[raw/sources/模型原理/模型架构/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了#GRM 与 Memory Soup 何时等价|原文：等价条件]]）

## 设计取舍

Memory Caching 把分段器、段内 RNN 与跨段聚合器解耦，可以接入 Linear Attention、Mamba、Titans 等不同记忆模块。相应代价是检查点缓存随段数线性增长，深度记忆模块的显存占用需要单独测量。

段长成为效率与召回之间的连续旋钮：短段保存更多检查点并提高检索成本，长段更接近固定容量递归记忆。论文只在段长为 1、特定向量值且无 Value 的记忆形式下推导门控全局注意力，不能推广为任意 Mamba／Titans 与 Transformer 等价。（[[raw/sources/模型原理/模型架构/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了#段长、容量与树状分段|原文：段长与复杂度]]）

论文比较等长分段与 Fenwick 树式对数分段；后者减少访问段数，使全序列复杂度含 $O(L\log L)$ 项，却让部分长历史压入单个状态。论文把该消融中的召回劣势归因于分辨率分配，不能推广为所有树状记忆必然更差。（[[raw/sources/模型原理/模型架构/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了#段长、容量与树状分段|原文：树状分段边界]]）

## 实验边界

原文核对论文 v1 Table 2 的 S-NIAH-3（UUID）16K 列：DLA 加 GRM 后从 4.0% 提高到 18.2%，Titans（LMM）从 21.2% 提高到 32.2%，均仍低于 Transformer 的 40.8%。另一套 Table 1 常识推理设置为 1.3B 参数、100B 训练 Token、4K 训练上下文和默认段长 256：Titans + GRM 为 58.33%，自身基线为 56.82%，Samba 为 54.46%，Transformer++ 为 53.19%。这些结果只属于对应模型与协议，不是其他系统的性能保证。（[[raw/sources/模型原理/模型架构/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了#论文实验支持什么|原文：实验设置与纠正数字]]）

## 关联

- 应用层上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 窗口、Attention 与 KV Cache：[[wiki/sources/上下文工程：第二期 窗口与 Token]]
- 上下文压缩与 KV 缓存优化边界：[[wiki/sources/上下文工程：第五期 上下文工程压缩]]
- Transformer 块级稀疏注意力：[[wiki/sources/模型架构：MoBA 混合块注意力]]
