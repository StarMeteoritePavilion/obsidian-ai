---
title: Codex里token是啥？文字是如何变成token的
source: https://www.bilibili.com/video/BV1LjTX6ZEJV
author: 张司机在路上
created: 2026-07-06
tags:
  - AI
  - Token
  - Tokenizer
  - BPE
  - 基础原理
  - 模型原理
---

# Codex里token是啥？文字是如何变成token的

Token 既不是固定的单个字符，也不是固定的完整单词。模型的上下文长度、API 成本和缓存命中都以 Token 计量，因此，理解文字怎样变成 Token，是理解这些数字的前提。

## Tokenizer 的两步工作

Tokenizer 可以粗略理解为执行两步转换。

第一步是把文本切成 Token。一个 Token 可能是完整单词、汉字、标点，甚至是 Emoji 的一部分。例如，“全民制作人们大家好”可以被切成“全民／制作／人／们／大家／好”。

第二步是把每个 Token 映射为词表中的数字 ID。大模型不会直接处理字符串，而是接收这些 Token ID。OpenAI 的 [Tokenizer 工具](https://platform.openai.com/tokenizer) 可以显示文本的切分结果，并切换到 Token ID 视图查看每个 Token 对应的数字。

## Codex 输出文本与计量 Token

资料用一次 GPT-5.5 的 Codex 对话比较可见文本的重新分词结果与响应 `usage` 中的 `output_tokens`：

- “Hello. What would you like to work on in the Codex repo?”被 Tokenizer 切为 15 个 Token，响应统计为 19 个输出 Token。
- “Got it. I’m here when you’re ready.”被切为 11 个 Token，响应统计为 15 个输出 Token。
- “You’re welcome.”被切为 4 个 Token，响应统计为 8 个输出 Token。

三组结果都相差 4。作者将这部分解释为系统开销。这个结果来自同一次对话中的三个响应，只能说明可见文本重新分词的数量不一定等于 API 返回的 `output_tokens`，不能据此认定所有模型、版本和响应都固定增加 4 个 Token。

模型输出本身是一串逐步生成的 Token ID，用户看到的是这些 ID 解码后的文字。将最终文字再次交给 Tokenizer 统计，是一次新的文本编码过程，不应脱离具体模型和请求格式推导固定差值。

## Tokenizer 在推理链路中的位置

输入阶段称为编码：Tokenizer 将文字转换为 Token ID；Token ID 只是离散编号，还要经过 Embedding，变成包含连续数值的高维向量，才能进入 Transformer。

推理完成后，模型输出一串 Token ID。Tokenizer 再把这些 ID 还原为用户可以阅读的文字，这个阶段称为解码。

因此，Token ID、Embedding 和 Transformer 内部表示是不同对象：Token ID 负责索引词表，Embedding 将离散编号映射为连续向量，Transformer 实际处理的是这些向量。

## 有限词表怎样覆盖任意文本

Tokenizer 依赖一张有限的 Vocabulary。它的任务是把输入文本表示成词表中已有 Token 的组合。词表怎样设计，会同时影响序列长度、覆盖能力、语义完整性和 Embedding 的参数规模。

### Word-based：以完整单词为 Token

Word-based 方法按空格和标点切分英文。例如，`I love you` 会成为三个 Token。

完整单词保留了较完整的语义，序列也较短；但遇到词表之外的新词时，只能使用 `[UNK]` 表示 Unknown Token，原词信息随之丢失。为了减少这种 Out of Vocabulary（OOV）问题，词表必须收录时态、单复数和派生词，规模可能增长到数十万甚至上百万，Embedding 也会占用更多显存。

### Character-based：以字符为 Token

Character-based 方法把每个字符作为一个 Token。英文只需覆盖字母、大小写和标点，中文也可以用数千个常用汉字形成较小的词表。新词和错别字仍能拆成字符表示，因此不会像整词方案那样因为新词直接落入 `[UNK]`。

代价是序列变长。`unpredictable` 按字符切分会形成 13 个 Token；在资料讨论的标准全注意力模型中，序列增长会增加 Attention 计算。单个字母通常也不具备完整词义，模型还需要学习哪些字符组合在一起才有意义。

### Subword：在两种极端之间折中

主流 Tokenizer 使用 Subword：高频常见词可以整体保留，罕见词则拆成更小的片段。例如，`unpredictable` 可以拆成 `un／predict／able`。Token 因而既可能是完整单词，也可能是词根或更小的片段。

这种方法在整词方案的大词表与字符方案的长序列之间折中，同时让罕见文本仍能由多个较小单元表示。

## BPE 怎样建立和应用词表

OpenAI 的 `tiktoken` 使用 Byte Pair Encoding（BPE，字节对编码）。资料把它概括为一种不断合并高频相邻单元的贪心算法，分为训练与推理两个阶段。

训练阶段先把语料拆成较小单元，统计最常相邻出现的组合，把最高频的一对合并成更大的 Token 并加入词表。系统随后继续统计、合并，重复多轮。经常共同出现的组合会逐层形成词根、半个单词或完整单词，每次合并的先后顺序则记录在规则表中。

推理阶段先把新文本拆成较小单元，再严格按照已经确定的规则顺序重放合并。训练时常见的组合会合并成较大的 Token；没有形成独立 Token 的罕见新词仍可以由多个小单元表示。BPE 由此在词表规模与未知文本覆盖之间取得折中。

## Tokenizer 与模型绑定

工程实践中会先训练 Tokenizer 并固定词表，再让大模型配合这套词表训练。Tokenizer 与模型由此形成绑定关系，不能随意互换。

不同模型可能使用不同的 Tokenizer，词表大小和切分规则也可能不同。同一段文字在 Codex 与 Claude Code 中得到不同的 Token 数，并不矛盾；比较上下文长度或计费时，必须以实际模型使用的 Tokenizer 和 API 统计为准。OpenAI 已将 `tiktoken` 开源，可从其代码继续查看实现细节。
