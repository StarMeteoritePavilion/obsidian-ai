---
title: 长序列建模：Memory Caching
source: https://www.bilibili.com/video/BV1GHP4zZESk
author: 唐国梁Tommy
published: 2026-03-06
ingested: 2026-09-02
updated: 2026-09-02
tags:
  - AI
  - 模型架构
  - 长序列建模
  - Memory Caching
  - 资料摘要
---

# 长序列建模：Memory Caching

原始资料：[[raw/sources/模型架构/长序列建模/RNN不再“金鱼记忆”！一篇论文讲透Memory Caching，Mamba长上下文终于能打了|RNN 不再“金鱼记忆”]]

## 核心问题

Transformer 通过 Attention 和随上下文增长的 KV Cache 保留完整历史，但需要承担较高显存与平方级计算代价；线性 RNN 将历史压缩到固定大小的隐状态，推理高效，却会在长序列中覆盖早期信息。Memory Caching 在 RNN 外部保存分段隐状态检查点，使历史记忆容量能够随段数增长。

这一问题位于模型架构层，不等同于应用层的上下文裁剪、RAG 或摘要压缩。二者都处理有限信息预算，但前者改变模型如何保存和查询序列状态，后者决定哪些外部信息进入当前窗口。

## 方法结构

序列按固定长度切段，每段由原有 RNN 处理；段末隐状态被冻结为检查点。生成当前 Token 时，系统同时使用段内在线记忆和历史检查点，并由跨段聚合器组合结果。

资料介绍四种策略：Residual Memory 等权聚合所有段；GRM 根据当前 Token 与各段摘要的相关性动态分配权重；Memory Soup 插值各段记忆模块的参数后再查询；Sparse Selective Caching 只激活少量最相关检查点，将超长历史下的实际检索量限制在固定范围。

GRM 的英文全称在资料内部不一致：旁白为 Gated Residual Memory，画面为 Gated Recurrent Memory。本库保留缩写，不合并这两个原始表述。

## 设计取舍

Memory Caching 把分段器、段内 RNN 与跨段聚合器解耦，可以接入 Linear Attention、Mamba、Titans 等不同记忆模块。相应代价是检查点缓存随段数线性增长，深度记忆模块的显存占用需要单独测量。

段长成为效率与召回之间的连续旋钮：短段保存更多检查点并提高检索成本，长段更接近线性 RNN。资料把段长为 1 视为趋近 Attention 的极端情况。

## 实验边界

资料转述的 16K Needle-in-a-Haystack 设置中，DLA 从 4% 提高到 18.2%，Titans 从 21.2% 提高到 32.2%，Transformer 为 40.8%。1.3B 常识推理实验中，Titans + GRM 为 58.33%，高于 Samba 的 56.12% 和 Transformer 的 53.19%。这些结果属于论文的特定模型、规模和任务，不能作为其他系统的性能保证。

## 关联

- 应用层上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 窗口、Attention 与 KV Cache：[[wiki/sources/上下文工程：第二期 窗口与 Token]]
- 上下文压缩与 KV 缓存优化边界：[[wiki/sources/上下文工程：第五期 上下文工程压缩]]
