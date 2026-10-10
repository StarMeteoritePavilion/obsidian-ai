---
title: 模型架构：从词嵌入到 RNN
source: https://www.bilibili.com/video/BV1NCgVzoEG9?p=5
author: 飞天闪客
published: 2025-07-21
ingested: 2026-10-09
updated: 2026-10-10
tags:
  - AI
  - 词嵌入
  - 词向量
  - RNN
  - 模型原理
  - 资料摘要
---

# 模型架构：从词嵌入到 RNN

原始资料：[[raw/sources/模型原理/基础原理/从词嵌入到 RNN|从词嵌入到 RNN]]

## 核心结论

词 ID 是标识，one-hot 高维且不同词正交；词嵌入以训练所得稠密向量表达关系，点积或余弦可作相似度度量。来源按列排列嵌入矩阵，不预设每维的自然语言含义。

RNN 每步接收一个词向量并更新隐藏状态。$W_{xh}$ 映射当前输入到隐藏状态，$W_{hh}$ 传递前一时刻状态，$W_{hy}$ 映射当前隐藏状态到输出。递归状态串联历史，也带来顺序计算和长程信息丢失问题。（[[raw/sources/模型原理/基础原理/从词嵌入到 RNN#隐藏状态把前一个词带下去|权重与公式]]）

## 例子里的数字

「他、开心、我、就、生气」共 5 个词，每词 300 维；整句直接展平会成为 1500 维输入，句长变化时输入宽度随之变化。10 万维 one-hot 以“词表有 10 万个词”为假设，不是已公布模型规格。桌子、椅子、鼠标、键盘的向量减法仅是教学示意。

## 与其他资料的边界

本篇只点名 word2vec、GRU、LSTM 与 Transformer，没有展开词向量训练、门控或注意力算法。[[wiki/sources/模型原理：词向量与嵌入矩阵|另一系列]]按矩阵行查词向量，方向约定与本篇分开；矩阵排布不构成语义或架构差异的实验证明。

## 关联

- [[wiki/sources/模型架构：简单而强大的 Transformer]]：下一 P 才展开注意力；本 P 仍然只点名 Transformer。
- [[wiki/sources/模型架构：从全连接矩阵到卷积神经网络]]
- [[wiki/sources/大语言模型：Token、Embedding 与 Latent Space]]
- [[wiki/sources/模型原理：从函数到神经网络]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
