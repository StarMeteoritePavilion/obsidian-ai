---
title: 模型架构：KV Cache 显存公式与 MHA、MQA、GQA
source: https://www.bilibili.com/video/BV1reKb6PEnw
author: 张司机在路上
published: 2026-07-21
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - KV Cache
  - Attention
  - MHA
  - MQA
  - GQA
  - 模型架构
  - 模型原理
  - 资料摘要
---

# 模型架构：KV Cache 显存公式与 MHA、MQA、GQA

原始资料：[[raw/sources/模型原理/模型架构/KV Cache真正在GPU上占用多少显存？多头注意力是什么？|KV Cache真正在GPU上占用多少显存？多头注意力是什么？]]

## 核心结论

KV Cache 保存的是每层 Attention 中已有 Token 的 Key 和 Value，不是原始聊天文字。对单个请求，资料给出的显存公式为：

$$
\text{KV Cache bytes}=2\times n\times d\times h_{kv}\times L\times \text{bytes per element}
$$

$n$ 是 Token 数，$d$ 是 Head Dimension，$h_{kv}$ 是 KV Head 数，$L$ 是 Decoder 层数，2 代表 Key 和 Value。上下文长度与 KV Cache 大小呈线性关系；MHA、MQA 和 GQA 的主要存储差异落在 $h_{kv}$。

## 三种 Attention 的存储取舍

- MHA 让每个 Query Head 保存独立 KV，因此 $h_{kv}$ 等于 Query Head 数。
- MQA 让所有 Query Head 共享一套 KV，因此 $h_{kv}=1$。
- GQA 将 Query Head 分组，组内共享 KV，使 $h_{kv}$ 介于两者之间。

资料将 MHA 概括为表达能力最强、MQA 概括为显存最省但质量下降，并将 GQA 视为两者的折中。该视频没有给出对应评测、模型配置或质量数字，因此这些表述不构成跨模型的固定排名。

## Qwen3-8B 计算案例

LMCache 页面所示 `Qwen/Qwen3-8B` 配置为 36 层、32 个 Attention Head、8 个 KV Head、Head Dimension 128，数据精度为 BF16。当 $n=100{,}000$ 时，计算得到 7,372,800,000 个 KV 元素、14,745,600,000 字节，页面标记为 13.7329 GB。

页面明示用 $1024^3$ 换算，因此 13.7329 按二进制单位实际对应 GiB；本摘要保留来源页面的 GB 标签并注明单位边界。该数字只属于这组模型、精度和上下文参数，不能作为其他模型的固定显存占用。

## 关联

- [[wiki/sources/模型推理优化：FlashAttention 算子融合、在线 Softmax 与 Tiling]]
- [[wiki/sources/模型架构：多头注意力与 QKV]]
- [[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]]
- [[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- [[wiki/syntheses/长上下文模型架构：共享、筛选、压缩与可增长记忆]]
