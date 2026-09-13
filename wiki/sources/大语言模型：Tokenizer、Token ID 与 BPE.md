---
title: 大语言模型：Tokenizer、Token ID 与 BPE
source: https://www.bilibili.com/video/BV1LjTX6ZEJV
author: 张司机在路上
published: 2026-07-06
ingested: 2026-09-13
updated: 2026-09-13
tags:
  - AI
  - Token
  - Tokenizer
  - BPE
  - 基础原理
  - 模型原理
  - 资料摘要
---

# 大语言模型：Tokenizer、Token ID 与 BPE

原始资料：[[raw/sources/模型原理/基础原理/Codex里token是啥？文字是如何变成token的|Codex里token是啥？文字是如何变成token的]]

## 核心结论

Tokenizer 先把文字切成 Token，再把每个 Token 映射为有限词表中的数字 ID。Token ID 只是离散索引，还需要经过 Embedding 变成连续向量，才能进入 Transformer；模型输出的 Token ID 则通过解码重新变成人类可读文本。

Word-based 切分保留完整词义、序列短，却面临 OOV、`[UNK]` 和大词表；Character-based 词表小、能组合新词，却会拉长序列并拆散语义。Subword 在两者之间折中，BPE 通过反复合并高频相邻单元建立词表，并在推理时按固定顺序重放合并规则。

## Codex 计量案例

资料中的三个 GPT-5.5 响应分别出现 `15→19`、`11→15` 和 `4→8`：左侧是对可见输出文本重新分词的数量，右侧是响应 `usage.output_tokens`。作者将恒定差 4 解释为系统开销，但这只是同一次对话的三个样本，不证明所有模型、版本和请求都固定增加 4 个 Token。

该案例补充了上一期资料的完整请求计量：系统规则、运行时注入、项目上下文和工具定义决定输入成本；模型输出计量也不能只用最终可见字符串重新分词来替代 API 统计。

## 适用边界

- Token 不是固定的字符或单词；实际边界由模型绑定的 Tokenizer 与词表决定。
- 视频用标准全注意力模型说明字符级长序列的计算代价，不能把平方复杂度不加区分地套到所有稀疏、线性或混合注意力实现。
- Word-based、Character-based 与 Subword 是教学对比；实际 Tokenizer 还取决于字符编码、预分词规则、特殊 Token 和具体词表。
- Codex 与 Claude Code 的 Token 数差异来自作者对不同 Tokenizer 的概括；具体计量应以实际模型和 API 返回为准。

## 关联

- [[wiki/sources/大语言模型：Token、Embedding 与 Latent Space]]
- [[wiki/sources/Codex：请求结构、服务端通信与 Token 计量]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- [[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
