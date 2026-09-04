---
title: 大语言模型：Token 与两类 Embedding
source: https://www.bilibili.com/video/BV1oNv8BPE2m
author: 隔壁的程序员老王
published: 2026-01-08
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型原理
  - 大语言模型
  - Token
  - Embedding
  - RAG
  - 资料摘要
---

# 大语言模型：Token 与两类 Embedding

原始资料：[[raw/sources/模型原理/大语言模型/15分钟弄懂Token和Embedding -- 详解LLM与RAG的数据处理机制|15分钟弄懂Token和Embedding -- 详解LLM与RAG的数据处理机制]]

## 核心结论

文字进入大语言模型前先由固定的 Tokenizer 转换为离散 Token ID，再由可训练的 Token Embedding 映射为连续向量。RAG Embedding 则由单独训练的模型把整段文本压缩为语义向量。二者可以使用相近组件，但粒度、位置和训练目标不同，不能视为同一个 Embedding。

## Tokenization 与向量入口

Token 不必对应完整单词。资料用 `old` 与 `est` 的组合说明子词切分怎样处理未见词，并以 GPT-2 的 `50,257` 词表解释中文覆盖不足带来的效率问题。资料还以 `tiktoken` 的 `o200k_base` 为例，称 GPT-4o 约 20 万规模的词表提高了中文覆盖。Tokenizer 属于固定预处理，不随大语言模型训练更新。

单个 Token ID 只承担索引作用，无法直接表达同义、语言、类别、颜色和形状等多重关系。Token Embedding 把编号映射到高维连续空间；真实向量由训练形成，资料中的可读特征轴只是教学类比，不能据此逐维解释模型参数。

资料给出的 GPT-2 XL Embedding 维度为 `1,600`，DeepSeek R1 为 `7,168`。画面同时列出未公开维度的 Gemini 3 和 GPT-5；作者对“上万维”的判断属于推测，不是公开规格。

## One-Hot、Linear 与输出映射

以 `50,257→1,600` 为例，Embedding 在数学上可以表示为 One-Hot 向量经过 Linear 映射。工程实现通常直接查表或使用稀疏矩阵，不必显式构造 One-Hot。Transformer 处理连续表示后，再通过输出 Linear 映射到词表空间；资料把这一步类比为 Embedding 的逆向操作。

## 两类 Embedding 的训练目标

大语言模型通过下一个 Token 预测学习生成；RAG Embedding 模型通过正负文本对学习语义距离，并通常使用 Encoder-only／BERT 路线和 Pooling 汇总整段文本。资料用“老王爱吃苹果”与“小王爱吃香蕉”作为正样本，用“老李找到了女朋友”作为负样本，说明对比学习分别拉近和拉远文本向量。

资料将大语言模型末端、尚未映射回 Token 的输出表示与 RAG Embedding 作直观类比，但同时明确两者必须分别训练。因此，更稳妥的综合关系是：它们都属于连续表示，却服务于不同目标，不能把任意 LLM Hidden State 直接等同为可用的检索向量。

## 系列定位与关联

官方“从0开始一起学大模型”合集将本资料列为第六条；视频完整解释 Tokenizer、Token Embedding 与 RAG Embedding，没有使用“第六期”标识，因此本库记录合集顺序，不另行命名期数。

- 系列导论：[[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- Linear 映射基础：[[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- 后续注意力专题：[[wiki/sources/模型架构：多头注意力与 QKV]]
- Token 与 Latent 综合：[[wiki/sources/模型原理：Token Space 与 Latent Space]]
- RAG 检索链路：[[wiki/sources/上下文工程：RAG 个人知识库基础架构]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
