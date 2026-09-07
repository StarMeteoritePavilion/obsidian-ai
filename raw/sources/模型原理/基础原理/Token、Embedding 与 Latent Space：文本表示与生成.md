---
title: Token、Embedding 与 Latent Space：文本表示与生成
source:
  - https://www.bilibili.com/video/BV1oNv8BPE2m
  - https://www.bilibili.com/video/BV1ABRyBqEoR
author:
  - 隔壁的程序员老王
  - 唐国梁Tommy
created: 2026-01-08
tags:
  - AI
  - 模型原理
  - 大语言模型
  - Token
  - Embedding
  - Latent-Space
  - RAG
updated: 2026-09-07
---

# Token、Embedding 与 Latent Space：文本表示与生成

大语言模型以离散Token作为输入和输出接口，主要计算则发生在连续向量中。完整链路可以写成：

> 文本 → Tokenizer → Token ID → Token Embedding → 多层隐藏状态 → logits → 下一个Token

RAG也使用Embedding，但它通常把一段文本编码为便于检索的向量，与生成模型输入端逐Token查表得到的Token Embedding用途不同。

## Tokenizer与Token ID

Tokenizer按照固定词表把字符串切成Token，再映射为整数ID。Token不必对应完整字或单词；同一个词可以由多个词片段组成，同一字符串在不同Tokenizer中也可能得到不同切分。

Token ID只是离散索引。编号本身没有“距离越近、含义越近”的保证，因此模型先通过Embedding矩阵把每个ID映射为连续向量。工程实现通常直接查表；数学上，这与将One-Hot向量乘Embedding矩阵等价，但无需显式构造巨大的One-Hot向量。

## Token Embedding、隐藏状态与logits

Token Embedding是每个输入位置进入模型时的初始向量。位置编码或旋转位置机制加入顺序信息后，Transformer层反复通过Attention和MLP更新每个位置的表示。层间主干通常称为Residual Stream；某层某位置的输出称为Hidden State。

最后一个相关位置的Hidden State经过输出投影，得到词表中每个Token的logits。Softmax可把logits转换为概率分布，温度、Top-k、Top-p和约束解码再决定如何选出下一个Token。logits、概率与最终采样结果是三个不同对象。

“Latent Space”在技术讨论中常泛指模型内部连续表示，但它不是一个单独房间。至少要说明以下位置：

| 表示 | 所在位置 | 是否跨上下文变化 | 主要用途 |
| --- | --- | --- | --- |
| Token Embedding | 模型输入端 | 查表结果固定，叠加位置后随位置变化 | 提供初始表示 |
| Hidden State／Residual Stream | 各Transformer层与各Token位置 | 是 | 承载上下文计算结果 |
| Attention／MLP Activation | 层内中间计算 | 是 | 搬运或加工信息 |
| KV Cache | 生成时保存的历史Key、Value投影 | 是 | 避免重复计算历史投影 |
| logits | 输出投影后的词表分数 | 是 | 选择下一个Token |

KV Cache是注意力实现中的历史投影缓存，不能与一般Hidden State或“潜在空间”视作同一种对象。

## Token Embedding与RAG Embedding

| 对比项 | Token Embedding | RAG Embedding |
| --- | --- | --- |
| 输入粒度 | 单个Token ID | 句子、段落或文档片段 |
| 输出粒度 | 每个Token一个初始向量 | 通常每段文本一个向量 |
| 训练目标 | 随生成模型的语言建模目标学习 | 常用对比学习或检索目标学习 |
| 上下文聚合 | 进入Transformer后才形成上下文化表示 | 编码器内部聚合，再通过Pooling等得到段落向量 |
| 用途 | 模型内部生成计算 | 相似度检索、聚类或召回 |

RAG Embedding模型不统一等于BERT或Encoder-only架构，也不统一禁用因果Mask；具体结构应以所用模型文档为准。向量归一化、Pooling方式和相似度函数也会改变检索结果。

## Tokenizer为何会影响任务表现

切分方式决定模型首先看到的离散单元，也决定同一文本占用多少上下文。英文空格、中文字符、数字分组、代码标识符和罕见符号可能形成不同粒度的Token。切分不理想会增加序列长度，也可能把任务所需的结构拆散。

因此，Token数量不能只按字符数估计；模型评测也应记录所用Tokenizer和提示文本的精确字节。原视频提到前导空格、数字格式等细微变化可能改变结果，但本文没有取得对应实验原件，不保留“最高11%”等精确幅度。

## 三类替代固定分词的路线

原视频列举了三种研究方向，它们解决的问题不同：

- **BLT**从Byte输入出发，根据局部预测难度动态组成Patch，把切分粒度纳入模型计算。
- **SuperBPE**允许形成跨空格的更长单元，以减少序列长度并学习多词结构。
- **T-Free**使用字符n-gram的稀疏表示，降低对固定词表中单个Token Embedding表的依赖。

这些路线不能只按Token减少比例排序。还要比较总训练计算、推理延迟、词表或哈希冲突、多语言表现与实现复杂度。原视频中的模型规模和降幅没有在本轮逐项取得论文版本，因此本文保留机制概览，删除精确数字。

## 可解释性：从特征到计算路径

模型需要表达的特征可能多于单层可直接分离的维度，多个稀疏特征会在激活空间中叠加。Sparse Autoencoder尝试把Hidden State重构为较稀疏的特征组合；重构误差与稀疏度之间存在权衡，抽出的特征也不能自动等同于稳定的人类概念。

Circuit Tracing继续研究特征之间的因果计算路径。线性探针、SAE解码或可视化只能提供观察；只有干预后行为按预测改变，才为因果解释增加证据。内部激活模式也不是模型具有情绪、人格或意识的证据。

## Latent Reasoning：隐藏步骤仍然需要计算

传统Chain of Thought把中间步骤写成自然语言Token，便于阅读、监督和审计，但会增加可见输出长度。Latent Reasoning尝试把部分中间计算保留在连续状态中。

- **Coconut**把某一步最后的Hidden State作为连续思考状态，再送回后续计算，不立即解码为文字。
- **Soft Thinking**用词表概率加权Embedding形成连续输入，暂时保留多个候选方向，而不是先采样一个离散Token。

两类方法的数据流都仍包含额外前向计算。减少可见Token不等于减少总FLOPs或真实延迟，也会降低逐步审计能力。原视频列出的pass@1和Token降幅缺少本轮可定位的论文版本，因此不作为本文结论。

## 评测与使用边界

仅观察最终回答，无法确定错误发生在Tokenization、内部表示、知识检索、指令遵循还是采样阶段。“模型知道但没有说”不能由输出文本单独证明。

比较不同表示或推理方式时，至少应固定模型版本、Tokenizer、提示字节、随机种子或采样设置，并同时记录准确率、可见Token、隐藏步骤、FLOPs和端到端延迟。模型公开上下文长度与Token价格属于容量和服务问题，应在对应专题中讨论，不与本文的表示层级混为一谈。

## 来源

| 来源 | 支持范围 |
| --- | --- |
| [Token与Embedding原视频](https://www.bilibili.com/video/BV1oNv8BPE2m) | Tokenizer、Token ID、Token Embedding、One-Hot等价关系与RAG Embedding对照 |
| [Token Space与Latent Space原视频](https://www.bilibili.com/video/BV1ABRyBqEoR) | 内部表示、分词研究、可解释性与Latent Reasoning讲解线索 |
| [Attention Is All You Need](https://arxiv.org/abs/1706.03762) | Transformer的Embedding、Attention、FFN与输出概率结构 |

本文删除了未公开的Gemini 3、GPT-5向量维度猜测、人生类比、未来路线预测，以及没有取得对应论文版本的精确提升幅度。
