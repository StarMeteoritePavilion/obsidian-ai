---
title: Transformer如何成为AI模型的地基
source: https://www.bilibili.com/video/BV1k6yWBEEmH
author: 隔壁的程序员老王
created: 2025-10-30
tags:
  - AI
  - 模型架构
  - Transformer
  - Encoder
  - Decoder
---

> “从0开始一起学大模型”合集首篇／Transformer 架构导论

2017 年，Google 发布论文《Attention Is All You Need》，介绍 Transformer 架构。2018 年，Google 基于 Transformer 设计的 BERT 在当时的多项 AI 任务中取得领先结果；2019 年，OpenAI 发布基于 Transformer 的 GPT-2，推动“大语言模型”进入更广泛的公众视野。此后，GPT、Gemini、Claude、DeepSeek 等模型都继续沿着 Transformer 路线发展。

Transformer 与这些模型的关系，可以先从最初的文本翻译结构入手。原始 Transformer 同时包含 Encoder 和 Decoder：Encoder 把原文转换为内部数字表示，Decoder 再依据这组表示逐个生成目标语言 Token。后来的模型则根据任务需要，分别发展出 Decoder-only 和 Encoder-only 等架构。

## Encoder：把输入转换为内部表示

原始 Transformer 用于文本翻译。以把 `I'm 王` 翻译为“我是王”为例，原文首先进入左侧 Encoder。资料把 Encoder 的输出类比为一张“含义矩阵”：它不对应某种人类语言，却以数字形式保存模型从原句中提取的信息。

“含义矩阵”是帮助理解的教学说法。关键在于，Encoder 接收完整输入，并把它转换为后续模块可以继续处理的内部表示。

Encoder 图中的“×N”表示相同模块结构会堆叠多层。每层结构相同，参数却彼此独立。若把某层简化为 $ax+b$，第一层可能使用 $2x+3$，第二层可能使用 $7x+1$；其中的 $a$ 和 $b$ 就是模型参数。训练模型，本质上是通过自动化过程持续调整这些参数，使计算结果逐渐符合目标。

资料提到，外界曾流传 GPT-4 拥有 1.8 万亿个参数，但这一数字没有被 OpenAI 正式确认。下载开源模型时，数十 GB 或上百 GB 的模型文件，主要保存的就是训练得到的大量数值参数。

## Decoder：逐个生成目标 Token

Decoder 接收两类输入：Encoder 产生的内部表示，以及当前已经生成的目标文本。生成开始时还没有目标文本，因此先输入句子开始标记。Decoder 根据开始标记和内部表示，为下一个 Token 计算概率分布。

例如，第一个位置可能给“我”分配 10% 概率，给“你”分配 4%，给“他”分配 5%，给“苹果”分配 0.5%。资料借 GPT-2 的 50257 个 Token 说明词表规模：模型会为词表中的每个 Token 给出一个分数或概率，再按解码策略选出下一个 Token。

选出“我”以后，开始标记和“我”共同成为下一轮输入，Decoder 再生成“是”；随后输入开始标记、“我”和“是”，继续生成“王”；最后生成结束标记，程序便知道翻译完成。

在整个过程中，Encoder 提供的内部表示保持不变，Decoder 侧已经生成的 Token 序列持续增长。Encoder 可以并行处理完整输入，Decoder 则按自回归顺序逐 Token 生成。资料据此解释，输出越长，解码计算和等待时间通常越高，也是许多大模型 API 将输出 Token 定价高于输入 Token 的原因之一。

## Temperature 与 Top-K

如果每次都选择概率最高的 Token，相同输入会倾向得到相同输出。为了增加生成的多样性，可以调整 Temperature，让较低概率的 Token 也获得被选中的机会。Temperature 越高，采样随机性通常越强；设为 0 时，常见实现会采用确定性选择。

还可以用 Top-K 限制采样范围。`K=1` 时只保留概率最高的 Token，`K=2` 时只在概率最高的两个 Token 中选择。字幕把这一参数转写为 `top-p`，但其描述的是按固定数量保留 Token 的 Top-K；Top-P 则按累计概率阈值确定保留范围。

## 原始翻译模型采用监督学习

训练原始翻译模型需要成对的原文和译文，例如 `I'm 王` 与“我是王”。Encoder 接收原文，Decoder 接收开始标记，训练目标要求它输出“我”；下一步再输入开始标记和“我”，训练模型输出“是”，依次完成整句翻译。

这种训练同时提供输入和期望输出，属于监督学习。模型并不是在训练时自由决定正确译文，而是根据成对样本不断调整参数，使每一步的预测更接近目标 Token。

## Decoder-only：从翻译模型到 GPT

原始 Transformer 的 Encoder 负责处理输入，Decoder 负责生成输出。GPT 系列沿着 Decoder 侧发展，保留适合自回归生成的主体，并去掉依赖独立 Encoder 输出的部分，形成 Decoder-only Transformer。

Decoder-only 模型适合文字接龙。输入“很高兴见”后，模型预测下一个 Token“到”；再把“很高兴见到”作为已有序列继续处理，预测“你”。资料将 GPT、Claude、Gemini、DeepSeek 和 Kimi 等主流生成模型概括为 Decoder-only 架构的不同变体。

这类模型还可以直接从普通文本构造训练目标。以“我是老王”为例，输入“我”并预测“是”，输入“我是”并预测“老”，输入“我是老”并预测“王”。训练材料只需提供有意义的文本，输入与目标可按照下一个 Token 预测规则自动形成，因此属于自监督学习。

## Encoder-only：BERT 的理解路线

Encoder 也可以独立使用。Google 发布的 BERT 采用 Encoder-only 架构，主要用于理解和提取文本信息，而不是逐 Token 生成长文本。

资料以“我是老王”为例：把中间的“是”替换为 `[MASK]`，形成“我 `[MASK]` 老王”，再训练模型恢复被遮住的内容。模型需要综合两侧上下文判断缺失 Token，因此能够学习双向文本表示。

这种表示可以服务于实体提取和其他文本标注任务。它没有 Decoder-only 生成模型那样通用的长文本生成方式，但在特定理解任务中仍有价值。

## Transformer 是共同地基，而不是单一成品

Transformer 最初以 Encoder—Decoder 结构解决翻译问题，后来分化出面向自回归生成的 Decoder-only 路线和面向双向理解的 Encoder-only 路线。它们共享分层参数计算和注意力等基础机制，但输入方式、训练目标与输出方式不同。

资料最后用人生阶段类比 Encoder 与 Decoder：前一阶段通过学习、经历、阅读和思考形成对世界的内部理解，后一阶段再把这些理解转化为表达、创作、传承和行动。这个比喻不属于模型机制，却概括了“先形成表示，再产生输出”的直观关系。

本篇只建立 Transformer 的整体结构。合集后续内容继续拆解 Linear、Activation、MLP、Attention 等内部组件。
