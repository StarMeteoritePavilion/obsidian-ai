---
title: 模型原理：Token Space 与 Latent Space
source: https://www.bilibili.com/video/BV1ABRyBqEoR
author: 唐国梁Tommy
published: 2026-05-05
ingested: 2026-09-03
updated: 2026-09-04
tags:
  - AI
  - 模型原理
  - 大语言模型
  - Token
  - Latent-Space
  - 资料摘要
---

# 模型原理：Token Space 与 Latent Space

原始资料：[[raw/sources/模型原理/大语言模型/Token Space 与 Latent Space：大模型如何处理与生成文本|Token Space 与 Latent Space：大模型如何处理与生成文本]]

## 核心结论

大语言模型通过 Token 接收和输出离散符号，主要计算则发生在连续隐藏表示中。最小生成路径是“文本 → Token → Embedding → Hidden State → logits → 下一个 Token”。Token Space 与 Latent Space 不是相互替代的机制，而是同一系统的接口层和计算层。

这一分层也用于定位失败：模型“没有编码相关信息”指向内部表示问题；“已经编码但没有输出”还可能来自解码、指令或安全策略；同一提示输出不稳定，则可能来自多个 Token 的 logits 接近与采样噪声。只观察最终文本，不能直接确定问题发生在哪一层。

## Tokenization 的影响

Tokenizer 决定模型收到的离散片段，切分边界会影响初始表示、内部语义几何和下游输出。资料列举中文语义部件、答案前导空格和数字分组等案例；其中多项选择题的 Token 边界差异在相应研究中使准确率波动最高 11%。

资料介绍三条新路线：BLT 根据 next-byte entropy 动态组成 Byte Patch；SuperBPE 允许跨空格形成更长的 superword，资料画面给出的结果为 Token 数减少 33%、推理算力降低 27%；T-Free 使用 character triplet 的稀疏表示替代传统 Token Embedding，资料画面给出的结果为 Embedding 体积减少 85%。相关节省比例均来自各自实验设置，不能直接外推。

## 内部表示与可解释性

Latent Space 不是单一位置。Embedding、Residual Stream、Attention 输出、MLP Activation、Hidden State 和 KV Cache 都是不同阶段的连续表示。Representation Engineering 尝试读出或干预概念方向，但可解释方向必须注明层、Token 位置和表示位置，并检验跨提示、模型与任务的稳定性。

Superposition 和 Polysemanticity 说明内部 Feature 会发生叠加，一个 Neuron 通常不能稳定对应一个概念。SAE 尝试把混合 Hidden State 拆成更易解释的 Feature，Circuit Tracing 则继续追踪 Feature 如何通过内部计算关系影响输出。内部激活模式不构成模型具有主观体验的证据。

## Latent Reasoning 的取舍

CoT 把推理写成可见 Token，便于监督和审计，但会增加生成成本，也不保证忠实反映内部计算。Coconut 将最后 Hidden State 作为 continuous thought 反馈给模型，资料称其在部分逻辑任务中表现出类似 Breadth-First Search 的行为；Soft Thinking 使用 Token 概率分布加权 Embedding，形成连续的 abstract concept token，资料画面给出的结果是相对标准 CoT，pass@1 最高提高 2.48%、Token 使用量最高减少 22.4%。

少生成可见 Token 不等于减少总计算。比较 Latent Reasoning 必须同时衡量 FLOPs、实际延迟、latent step、训练数据和评测偏向。未来 Token-Latent Hybrid 可能保留 Token 作为人机交互、工具调用与审计接口，把部分规划、搜索和压缩放入 Latent Space；相应代价是隐藏计算更难解释和追责。

## 关联

- Tokenizer 与两类 Embedding：[[wiki/sources/大语言模型：Token 与两类 Embedding]]
- 推理表示综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- 多模态交错推理：[[wiki/sources/多模态推理：ThinkMorph 交错思维链]]
- Token、窗口和 KV Cache：[[wiki/sources/上下文工程：第二期 窗口与 Token]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 长序列隐藏状态与记忆：[[wiki/sources/长序列建模：Memory Caching]]
