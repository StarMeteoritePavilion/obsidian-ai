---
title: 模型推理优化：KV Cache 与 Prompt Cache 的复用层级
source: https://www.bilibili.com/video/BV1DsG76AEEc
author: 张司机在路上
published: 2026-05-24
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Transformer
  - Attention
  - KV Cache
  - Prompt Caching
  - Prefill
  - Decode
  - 推理优化
  - 模型工程
  - 资料摘要
---

# 模型推理优化：KV Cache 与 Prompt Cache 的复用层级

原始资料：[[raw/sources/模型工程/推理与服务/提示词缓存里到底存了什么？和KV Cache有什么区别？|提示词缓存里到底存了什么？和KV Cache有什么区别？]]

## 核心结论

KV Cache 与 Prompt Cache 都复用 Transformer 的 Key／Value 中间状态，但作用范围不同：KV Cache 在单次请求内避免 Decode 反复计算历史 K／V；Prompt Cache 把 Prefill 后的稳定前缀 K／V 保留到后续请求。Prompt Cache 命中不会返回历史答案，本次后缀和输出仍需计算。

## 从 Attention 到缓存

资料使用标准缩放点积注意力：

$$
\operatorname{Attention}(Q,K,V)
=\operatorname{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

对 $n$ 个 Token、$d$ 维向量，$Q$ 为 $n\times d$，$K^T$ 为 $d\times n$，$V$ 为 $n\times d$，分数矩阵为 $n\times n$。Prefill 批量建立输入前缀的 K／V；Decode 每步产生新的 Query、Key、Value，只需把新 K／V 追加到缓存，并让新 Query 读取历史 K／V。

Query 只在当前生成步骤使用一次，Key 与 Value 则会被后续每一步重复读取，因此 KV Cache 保存 K／V 而不保存历史 Query。

## 复用层级

- **KV Cache**：当前请求的动态状态。Prefill 初始化输入 K／V，Decode 持续追加新 Token 的 K／V；请求结束后释放。
- **Prompt Cache**：跨请求的半持久状态。当前缀完全相同时，后续请求复用此前 Prefill 得到的 K／V，只计算新增后缀；资料称其按 LRU 策略过期。

资料把 KV Cache 的 Decode 单步复杂度概括为 $O(n^2)\rightarrow O(n)$，把 Prompt Cache 命中前缀的 Prefill 概括为 $O(n^2)\rightarrow O(1)$。这些是“避免重新计算对应历史部分”的教学口径，不是端到端延迟公式：缓存查询、显存读取、新增后缀、Decode、调度和其他算子仍然存在。

## 与 Claude Code 缓存资料的关系

Claude Code 的三轮抓包展示了计量结果：第二轮读取首轮写入的 48,654 Token，并新增写入 24 Token；本资料解释这些命中内容为什么不是历史答案，而是稳定前缀已经完成的 Prefill 中间状态。

另一份缓存实践资料称默认 TTL 为五分钟，并可选一小时 TTL；本资料最终对照则把 Prompt Cache 描述为按 LRU 过期。两者分别描述产品暴露的有效期和教学图中的淘汰策略，现有资料不足以把它们合并成 Anthropic 服务端的完整缓存实现。

## 证据边界

- “几千 Token 快几千倍、上万 Token 快上万倍”来自复杂度类比，没有端到端 Benchmark，不能作为实际延迟倍数。
- 资料表格把 KV Cache 写为“当前对话中已生成 Token”的 K／V；同一资料的流程图同时显示 Prefill 会先把输入 Token 的 K／V 写入缓存，因此更准确的资料内口径是“当前请求已经处理的输入与输出 Token”。
- Prompt Cache 的 $O(1)$ 表示命中部分不重新 Prefill，不表示完整请求、缓存读取或新 Token 生成没有成本。
- 跨请求保留、LRU 和显存位置是资料所示服务实现口径，不能外推为所有模型服务平台的固定方案。

## 关联

- Claude Code 多轮计量：[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本]]
- Claude Code 命中边界：[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- Codex 前缀缓存：[[wiki/sources/模型推理优化：Codex 自动前缀缓存]]
- 通用成本结构：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- KV Cache 显存：[[wiki/sources/模型架构：KV Cache 显存公式与 MHA、MQA、GQA]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
