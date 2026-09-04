---
title: 模型推理优化：Token 成本、KV Cache 与缓存机制
source: https://www.bilibili.com/video/BV1e17y6wEy5
author: 唐国梁Tommy
published: 2026-06-05
ingested: 2026-09-03
updated: 2026-09-03
tags:
  - AI
  - 模型工程
  - 推理优化
  - Token
  - KV-Cache
  - Prompt-Caching
  - 资料摘要
---

# 模型推理优化：Token 成本、KV Cache 与缓存机制

原始资料：[[raw/sources/模型工程/推理优化/同样100万Token，为什么有的模型贵30倍？大模型 API 到底在卖什么？一口气讲透 Token 成本、KV Cache 和缓存机制|同样100万Token，为什么有的模型贵30倍？]]

## 核心结论

Token 已从分词单位扩展为模型能力、GPU 推理、显存、缓存、延迟、调度和服务保障的商业计量单位。同样 100 万 Token 不一定是同一种商品：资料把高价模型的主要价值概括为复杂任务中的确定性，把低价模型的主要价值概括为标准任务中的规模化性价比。

## 成本如何形成

Prefill 可以并行处理完整输入，成本较容易摊薄；Decode 必须逐 Token 自回归生成，每步都需要计算和调度，通常更贵。用户不可见的 Reasoning Token 在资料描述的平台结构中也按输出计费。KV Cache 避免重复计算历史状态，却随上下文长度和并发增加显存占用，使吞吐、延迟与长上下文价格相互关联。

Prompt Caching 把可复用的稳定前缀转成低成本输入。资料称缓存命中价往往约为原价十分之一，部分平台宣称最多节省 90% 输入成本、降低 80% 延迟；具体收益取决于前缀长度、有效期、缓存断点和平台命中规则。Batch 则用等待时间换取更高 GPU 利用率，资料称价格通常约为标准价的一半。

## 应用架构含义

真实成本不能只看每百万 Token 单价，还应计入用量、延迟、错误、人工稽核、运维、重试和任务成功率。长输入短输出任务应优先治理上下文与缓存，长输出任务应控制生成长度；离线任务评估 Batch，复杂度不同的任务采用模型路由，并以 Token FinOps 持续观察高消耗用户、缓存命中率和每个成功任务的平均成本。

## 来源边界

价格倍率、长上下文档位、缓存节省上限和 Batch 折扣反映资料对视频发布时各平台规则的概括，不是跨平台固定价格。实际选型必须核对具体模型、区域、服务等级和当前官方价格页。

## 关联

- GQA 与 Sparse Attention 的长上下文优化：[[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]]
- Token 与内部表示：[[wiki/sources/模型原理：Token Space 与 Latent Space]]
- 推理执行优化：[[wiki/sources/模型推理优化：DSpark 投机解码]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
