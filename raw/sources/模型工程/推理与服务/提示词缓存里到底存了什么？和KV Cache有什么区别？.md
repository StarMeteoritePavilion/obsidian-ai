---
title: 提示词缓存里到底存了什么？和KV Cache有什么区别？
source: https://www.bilibili.com/video/BV1DsG76AEEc
author: 张司机在路上
created: 2026-05-24
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
---

# 提示词缓存里到底存了什么？和KV Cache有什么区别？

Prompt Cache 命中后，服务器不会直接返回某次历史回答。缓存保存的是相同前缀经过 Transformer Prefill 后得到的 K／V 中间状态；模型仍然需要处理本次新增内容，并继续执行 Decode 生成新的回答。

理解这一区别，需要先拆开 Prefill、Decode、Attention 与 KV Cache 的关系。

## Prefill 与 Decode

Transformer 推理分为两个相连阶段。

Prefill 先处理完整提示词，为输入 Token 建立后续生成所需的中间状态。即使提示词包含一千个 Token，这一阶段也会对整段输入进行批量计算。

Decode 随后以自回归方式逐 Token 生成答案。每生成一个 Token，模型都要结合完整提示词和此前已经生成的内容，决定下一个 Token。Prompt Cache 主要复用 Prefill 阶段已经完成的前缀计算。

## Attention 如何选择相关信息

Attention 并不把前文中的每个 Token 视为同等重要。资料以“喜欢／唱／跳／Rap／还有”预测下一个 Token 为例：对“唱”“跳”“Rap”的注意力权重更高，模型最终生成“篮球”。这个示例用于说明，模型根据当前 Query 与历史 Key 的匹配程度分配权重，而共同出现关系来自训练中学到的模型权重。

每个 Token 会被投影成三个向量：

- Query 表示当前 Token 想寻找什么；
- Key 表示该 Token 能提供什么信息；
- Value 表示该 Token 实际携带的信息。

当前 Query 与各个 Key 计算匹配分数，Softmax 把分数归一化为总和为 1 的权重，再用这些权重汇总各个 Value：

$$
\operatorname{Attention}(Q,K,V)
=\operatorname{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

其中，$QK^T$ 产生匹配分数，除以 $\sqrt{d_k}$ 用于缩放，Softmax 完成归一化，最后乘以 $V$ 得到加权结果。

## Prefill 的矩阵计算

设提示词包含 $n$ 个 Token，每个 Query、Key、Value 向量均为 $d$ 维。把各 Token 的向量按行排列后：

- $Q$ 的形状为 $n\times d$；
- $K^T$ 的形状为 $d\times n$；
- $V$ 的形状为 $n\times d$。

$QK^T$ 得到 $n\times n$ 的分数矩阵，表示输入 Token 之间的两两匹配分数。该矩阵再经缩放和 Softmax 与 $V$ 相乘。资料据此把 Prefill 的主要计算描述为随输入长度按平方增长。

## Decode 为什么需要 KV Cache

Decode 每生成一个新 Token，只产生该 Token 对应的一行 Query、Key 和 Value。新的 Query 需要与历史 Key 计算分数，再按权重读取历史 Value；随后，新 Token 的 Key 和 Value 被追加到已有矩阵末尾。

旧 Query 只在对应 Token 产生时使用一次，下一步会换成新的 Query，因此没有持续保存的必要。历史 Key 和 Value 一旦计算完成便保持不变，而且每个后续 Decode 步骤都需要重新读取，所以值得缓存。

KV Cache 就是把这些不变的 Key 和 Value 保存下来。Prefill 一次性计算输入前缀的 K／V 并初始化缓存；Decode 每生成一个 Token，只计算并追加新的 K／V，历史部分直接从缓存读取。

资料采用的教学复杂度口径是：如果每一步都从头重算历史表示，Decode 单步为 $O(n^2)$；使用 KV Cache 后，单步降为 $O(n)$。资料进一步用几千个 Token 对应几千倍、上万个 Token 对应上万倍作直观类比。这是复杂度层面的说明，不是包含访存、调度和其他算子的端到端速度实测。

## Prompt Cache 保存的是 Prefill 中间状态

完成 Prefill 后，输入前缀的 Key 和 Value 已经写入 KV Cache。Prompt Cache 保存的正是可跨请求复用的这部分 K／V 中间状态，而不是此前生成的自然语言回答。

没有 Prompt Cache 时，每个新请求都要重新 Prefill 相同的系统提示词、工具定义和历史对话。前缀完全相同时，Prompt Cache 可以取出此前已经计算好的 K／V，只计算本次新增的后缀，再进入正常 Decode。

缓存命中并不等于直接取得答案。它只跳过命中前缀的重复 Prefill；本次新增输入仍需计算，输出仍需逐 Token 生成。

## 两类缓存优化不同阶段

KV Cache 面向单次请求内的 Decode。它保存当前请求已经处理的输入与输出 Token 对应的 K／V，在生成过程中动态追加，并在该请求生命周期结束后释放。

Prompt Cache 面向跨请求的 Prefill。它以完全相同、可高频复用的前缀为匹配对象，保存对应 K／V，使系统提示词等静态长文本不必在每次请求中重复计算。资料把它描述为半持久状态：跨请求保留、命中时复用，并按 LRU 策略过期。

资料在最终对照中把 KV Cache 的 Decode 单步计算写为从 $O(n^2)$ 降至 $O(n)$，把 Prompt Cache 命中部分的 Prefill 写为从 $O(n^2)$ 降至 $O(1)$。这里的 $O(1)$ 表示命中前缀的 Prefill 被直接跳过，并不表示缓存查询、数据读取或整个请求没有任何成本。

两类缓存的共同载体都是 Key／Value 中间状态，但作用范围不同：KV Cache 消除同一次请求内 Decode 的历史 K／V 重算，Prompt Cache 消除跨请求 Prefill 的相同前缀重算。
