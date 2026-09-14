---
title: KV Cache真正在GPU上占用多少显存？多头注意力是什么？
source: https://www.bilibili.com/video/BV1reKb6PEnw
author: 张司机在路上
created: 2026-07-21
tags:
  - AI
  - KV Cache
  - Attention
  - MHA
  - MQA
  - GQA
  - 模型架构
  - 模型原理
---

# KV Cache真正在GPU上占用多少显存？多头注意力是什么？

Prompt Cache 复用的是提示词上下文对应的 KV Cache。这份缓存不是原始文字或聊天记录，而是每个 Token 在 Attention 中计算得到的 Key 和 Value。它的显存占用取决于 Token 数、每个 Attention Head 的维度、KV Head 数、Decoder 层数和每个元素的字节数。

## KV Cache 保存的是 Key 和 Value

Transformer 通过 Attention 在上下文中寻找与当前 Token 最相关的信息。Token 的 Embedding 被投影为 Query、Key 和 Value，其计算可写为：

$$
\operatorname{Attention}(Q,K,V)=\operatorname{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

在 Decode 阶段，每生成一个新 Token，模型都会产生新的 Query，再用它匹配所有旧 Key 并对旧 Value 加权求和。Query 每步都会更新，用完即可丢弃；历史 Key 和 Value 却会被后续每一步重复使用，因此需要持续保存。

假设提示词长度为 $n$ 个 Token，单个 Head 中每个 Token 的向量维度为 $d$，那么 Key 和 Value 矩阵都是 $n\times d$。单层、单组 KV 的缓存大小为：

$$
2\times n\times d\times \text{bytes per element}
$$

式中的 2 分别代表 Key 和 Value。FP16 的每个元素占 2 字节，因此此时 `bytes per element` 为 2。

## 多头注意力增加 KV 存储

一套 Attention 只能通过一组投影衡量 Token 之间的关系。资料以指代、动作的主动与被动关系，以及“苹果”是公司还是水果为例，说明语言中存在多种需要并行捕捉的关系。

Multi-Head Attention（MHA）将 Attention 分为多个 Head，每个 Head 使用自己的 Query、Key 和 Value 投影计算输出。各 Head 的输出沿水平方向拼接，再通过输出投影恢复为模型所需的维度。

如果共有 $h$ 个 Head，每个 Head 都保存独立的 Key 和 Value，KV Cache 大小便增加为：

$$
2\times n\times d\times h\times \text{bytes per element}
$$

## 每一层 Decoder 都有独立 KV

真实 Transformer 会堆叠多层 Decoder。每层通常包含 Masked Multi-Head Attention、Add & Norm、Feed Forward 和另一个 Add & Norm。Feed Forward 与 Add & Norm 帮助深层网络训练与收敛，本身不产生新的 Key 和 Value。

各层输入逐层更新，因此每层得到的 Key 和 Value 也不同。资料指出，若某层不保存历史 KV，生成时就需要重新计算该层的历史状态，使对应计算从 $O(n)$ 上升到 $O(n^2)$。因此，若 Decoder 共有 $L$ 层，全部层的 KV Cache 都需计入：

$$
2\times n\times d\times h\times L\times \text{bytes per element}
$$

## MQA 与 GQA 减少 KV Head

MHA 让每个 Query Head 使用独立的 Key 和 Value，表达能力较强，但每个 Head 都要保存一套 KV。Multi-Query Attention（MQA）保留多个独立 Query Head，却让所有 Head 共享一套 Key 和 Value。按资料的概括，这能大幅节省显存，但会削弱多头并行表示不同信息的能力。

Grouped-Query Attention（GQA）介于两者之间：Query Head 被分成多组，同组共享一套 Key 和 Value。八个 Query Head 两两分组时，只需保存四组 KV。MQA 相当于 KV Head 数为 1 的特例，MHA 则相当于 KV Head 数与 Query Head 数相同的特例。

用真正需要存储的 KV Head 数 $h_{kv}$ 替代 Query Head 数，最终公式为：

$$
\text{KV Cache bytes}=2\times n\times d\times h_{kv}\times L\times \text{bytes per element}
$$

## Qwen3-8B 的十万 Token 示例

LMCache 页面的 KV Cache 计算器以 `Qwen/Qwen3-8B` 为默认模型。画面所示参数为 36 层、32 个 Attention Head、8 个 KV Head、Head Dimension 为 128，数据精度为 BF16，每个元素占 2 字节。当上下文长度设为 100,000 Token 时：

$$
2\times36\times100{,}000\times8\times128=7{,}372{,}800{,}000\ \text{elements}
$$

$$
7{,}372{,}800{,}000\times2=14{,}745{,}600{,}000\ \text{bytes}
$$

计算器将字节数除以 $1024^3$，并将结果标为 **13.7329 GB**。

这个案例展示了 KV Cache 与上下文长度的线性关系：模型和数据精度不变时，Token 越多，单个请求保存的 KV 也越多。GQA 通过减少 $h_{kv}$，在不只保留一套 KV 的前提下缩小显存占用。
