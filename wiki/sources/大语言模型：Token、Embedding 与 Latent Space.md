---
title: 大语言模型：Token、Embedding 与 Latent Space
source:
  - https://www.bilibili.com/video/BV1oNv8BPE2m
  - https://www.bilibili.com/video/BV1ABRyBqEoR
author:
  - 隔壁的程序员老王
  - 唐国梁Tommy
published: 2026-01-08
ingested: 2026-09-03
updated: 2026-09-07
tags:
  - 基础原理
  - AI
  - 模型原理
  - 大语言模型
  - Token
  - Embedding
  - Latent-Space
  - RAG
  - 资料摘要
---
# 大语言模型：Token、Embedding 与 Latent Space

原始资料：[[raw/sources/模型原理/基础原理/Token、Embedding 与 Latent Space：文本表示与生成|Token、Embedding 与 Latent Space：文本表示与生成]]

## 核心结论

大语言模型以离散Token作为输入和输出接口，主要计算发生在连续表示中：Tokenizer把文本变为Token ID，Token Embedding提供初始向量，Transformer形成上下文化Hidden State，输出投影产生词表logits。Token Embedding、Residual Stream、层内Activation、KV Cache和logits位于不同计算位置，不能统称为同一个可互换的“Latent Space”。

RAG Embedding通常把句子或段落编码为单个检索向量，训练目标、聚合方式和用途均不同于生成模型的Token Embedding。具体RAG模型也不统一等于BERT或Encoder-only架构。

## 表示与推理边界

Tokenizer决定模型首先看到的离散结构与文本占用的上下文长度。BLT、SuperBPE和T-Free代表字节动态分块、跨空格长单元和字符n-gram等不同路线；当前原文因缺少对应论文版本，没有保留原视频中的精确节省比例。

SAE尝试把叠加的隐藏状态重构为稀疏特征，Circuit Tracing继续追踪特征间的计算路径。可解释方向需要注明层、Token位置与表示位置，并通过干预验证；内部激活不构成意识或人格证据。

Coconut和Soft Thinking把部分中间步骤保留在连续状态中，但隐藏步骤仍需要前向计算。减少可见Token不等于减少总FLOPs或延迟，也会降低逐步审计能力。

## 证据限制

仅从最终文本不能判断错误发生在分词、内部表示、检索、指令还是采样阶段，也不能证明“模型知道但没有说”。原视频提及的前导空格波动、模型维度和若干方法提升幅度因缺少本轮可定位原件，未进入当前结论。

## 关联

- [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- [[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- [[wiki/sources/模型架构：多头注意力与 QKV]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
