---
title: Token Space 与 Latent Space：大模型如何处理与生成文本
source: https://www.bilibili.com/video/BV1ABRyBqEoR
author: 唐国梁Tommy
created: 2026-05-05
tags:
  - AI
  - 模型原理
  - 大语言模型
  - Token
  - Latent-Space
---

> 大语言模型原理独立专题

大语言模型不是直接读取人类文字，也不是在一个神秘空间中直接“思考”。它接收和输出的是离散 Token，主要计算则发生在连续的隐藏表示中。完整路径是：文本被切分为 Token，Token 经过向量化和多层 Transformer 计算形成隐藏状态，隐藏状态再被投影为词表上的分数分布，最终选出下一个 Token。

## 从 Token 到 Latent，再回到 Token

Token 是模型处理文本的离散单位，不必对应完整的字或词。Tokenizer 先把输入切成 Token，并将其转换为词表中的 Token ID；Embedding Matrix 再把离散编号映射为连续向量，为每个 Token 提供初始表示。

Transformer 的各层继续处理这些向量。Attention 从上下文中读取和搬运信息，MLP 通过非线性计算加工信息，多层计算后形成 Hidden State。模型最后把 Hidden State 投影回整个词表，得到每个 Token 的原始分数 logits，再经过 Temperature、Top-K、Top-P 或受约束解码等策略选择下一个 Token。

最小生成流程可以写成：

> 文本 → Token → Embedding → Hidden State → logits → 下一个 Token

Token Space 是离散接口，承接输入、训练标签、输出和评测；Latent Space 是模型内部的连续表示空间，承接上下文理解、语义压缩、概念表征和中间计算。二者不是相互竞争的两种机制，而是同一系统的不同阶段。

这个区分也有助于分析模型失败。“模型不知道”可能意味着内部表示没有正确编码相关信息；“模型知道但没有说”可能来自解码、指令或安全策略；同一提示的回答不稳定，则可能与多个候选 Token 的 logits 接近及采样噪声有关。仅观察最终文本，无法直接判定问题发生在哪一层。

## Tokenizer 会改变模型接收的信息结构

Tokenizer 不改变文本对人的客观含义，却会改变模型收到的离散片段。英文存在明显空格边界，中文没有天然空格；当 Token 边界与语义单位、中文部首结构或其他语言的词缀结构不对齐时，进入 Latent Space 的初始信号会受到影响。资料据此强调，Token 边界不是无关紧要的工程细节，它会继续影响内部表示和输出选择。

正式评测也可能受到切分方式干扰。资料引用的多项选择题研究显示，仅改变 `Answer:` 后的前导空格，使答案字母成为 single-token 或 multi-token，就可能让准确率波动最高 11%，甚至改变模型排名。数字任务同样会因切分方式不同而形成完全不同的 Token 序列；资料展示的 `1234567`、`1,234,567` 和三位一组的表示，在对应实验中得到不同算术准确率。

同一字符串也不只存在一种 Token 序列表示。默认 Tokenizer 给出标准切法，但非标准序列仍可能表达相同字符串。模型能力并未完全被标准切法锁定，具体切分方式却会使某些任务更容易、另一些任务更脆弱。

## Tokenization 的三条新路线

Tokenizer 的发展方向并非简单地越细越好或越粗越好，而是把切分粒度交给更适合的机制。

### BLT：动态决定字节块粒度

Byte Latent Transformer（BLT）直接从 Byte 层处理输入，再动态组成 Patch。它依据 next-byte entropy 判断切分粒度：简单片段可以组成更大的 Patch，复杂片段则保留更细的建模粒度。固定 Tokenizer 预先决定文本怎样切分，BLT 则让模型在架构内部决定哪里细看、哪里粗看。

资料所述实验覆盖 8B 参数和 4T bytes，意在展示 Byte-level 模型追上传统 tokenizer-based LLM 的可行性。其意义不只是不使用固定 Tokenizer，还在于把切分粒度纳入模型计算。

### SuperBPE：形成跨空格的长 Token

SuperBPE 允许 Token 跨越空格形成更长的 superword。资料画面给出的结果是 Token 数减少 33%、推理算力降低 27%。这些结果属于相应实验设置，不代表所有模型和任务都能得到相同比例。

### T-Free：使用字符三元组

T-Free 使用 character triplet，即三个字符一组的稀疏表示，替代传统 Token Embedding。资料画面给出的结果是 Embedding 体积减少 85%，并指出该方法在多语言和低资源语言上显示出潜力。

三条路线共同说明，Tokenization 的本质不只是切词，而是在词表、上下文长度和计算预算受限的条件下，把语言信号编码成适合模型学习的离散表示。

## Latent Space 不是单一房间

Latent Space 是模型内部表示信息的连续向量空间。Hidden State 可能同时压缩语义、风格、任务、上下文关系和下一步输出倾向，但这些向量本身不能由人直接阅读。

模型内部也不存在一个唯一的 Latent Space。Embedding 是 Token 的初始向量入口；Residual Stream 是层间传递信息的主干；Attention 输出、MLP Activation 和生成时保存历史上下文的 KV Cache，都属于不同位置和阶段的连续表示。因此，当研究声称在 Latent Space 中发现某个方向时，还必须说明层、Token 位置和表示位置，并检验它能否跨提示、模型和任务保持稳定。

Representation Engineering 尝试从内部表示中读出并干预某些概念方向。已有研究发现空间、时间、情感和政治视角等概念可以被线性读出，这说明内部表示具有结构和几何关系；但单个坐标轴通常不等于一个完整的人类概念。更有意义的问题是，相对关系和方向能否稳定读出信息，干预后是否确实改变输出。

这些内部表示是人工神经网络的激活模式，不是模型具有真实情绪、人格、意识或主观体验的证据。

## 从叠加特征到内部计算路径

Mechanistic Interpretability 关注模型通过哪些内部计算步骤得到输出。其中一个核心问题是 Superposition：模型需要表示的 Feature 很多，可用维度有限，而且许多 Feature 稀疏出现，因此多个不常同时出现的特征可能被压入同一有限空间。

这种叠加会产生 Polysemanticity。同一个 Neuron 或 Activation Pattern 在不同上下文中可能关联多个看似无关的概念，所以“一个神经元等于一个概念”通常不稳定。概念更常表现为分布式模式，某次激活也可能混合多个特征。

Sparse Autoencoder（SAE，稀疏自编码器）试图把混合的 Hidden State 拆成更容易解释的 Feature 组合。资料引用 Anthropic 2024 年的《Mapping the Mind of a Large Language Model》，其中从 Claude 3 Sonnet 的中间层提取了数百万个可解释 Feature，并观察到对部分 Feature 的干预会改变模型行为。SAE 的主要用途是帮助理解内部激活，而不是直接增强模型能力，也不能被视为始终可靠的控制器。

Circuit Tracing 进一步追踪输入信息经过哪些内部 Feature 和计算步骤，最终影响某个输出。SAE 主要回答模型内部有哪些可解释特征，Circuit Tracing 则继续追问这些特征如何形成计算关系并导致输出。

## Latent Reasoning：不把每一步都写成 Token

传统 Chain of Thought（CoT）在 Token Space 中展开推理，把中间步骤写成自然语言。它便于阅读、监督、采样和审计，但会增加 Token 生成成本；自然语言带宽有限，已经写出的中间步骤也难以修改，而且可见 CoT 不一定忠实反映模型内部真正的推理过程。

Latent Reasoning 允许模型把中间推理保留在 Hidden State 中，只在最后输出答案或必要解释。其核心直觉是，中间推理不必每一步都重新落回词表。

Coconut 把 LLM 的最后 Hidden State 当作 continuous thought，不把中间状态解码为文字，而是直接作为下一步输入向量送回模型。资料称，这种连续思考可以同时保留多个推理方向，避免过早绑定到单一自然语言路径，并在部分逻辑任务中表现出类似 Breadth-First Search（BFS）的行为。

Soft Thinking 不立即采样离散 Token，而是用 Token 概率分布加权 Embedding，形成连续的 abstract concept token，暂时保留多个候选方向的混合表示。资料画面给出的实验结果是，相对标准 CoT，pass@1 最高提高 2.48%，Token 使用量最高减少 22.4%。这些数字只属于对应论文的实验设置。

减少可见 Token 不等于减少总计算，也不证明 Latent Reasoning 必然更聪明或高效。公平比较需要同时衡量总 FLOPs、wall-clock latency、latent step 数、训练数据和 Benchmark 偏向；隐藏步骤只是在另一处发生计算，不能被当作免费推理。

## Token-Latent Hybrid 与可审计性

未来架构更可能让 Token Space 和 Latent Space 分工，而非由一方取代另一方。人类交互、最终回答、工具调用、程序代码和安全审计仍需要 Token，因为它是可读、可监督、可评测、可记录的接口；中间推理压缩、多路径搜索、语义规划、跨语言表示和多模态融合则可能更多发生在 Latent Space。

一种可能的流程是：Token 输入进入 Latent Planning 与 Reasoning，形成 concept-level state，必要时调用工具和记忆，最后返回 Token 输出。对人可见的部分保留 Token，模型内部适合连续计算的部分使用 Latent State。

这种变化也带来三个开放问题：固定 BPE Tokenizer 是否仍是合理默认选择；Latent CoT 应如何解释和审计；模型评测是否应从可见 Token 扩展到隐藏计算。Decoder、Probe、SAE 和 Causal Intervention 可以协助研究隐藏状态，但这些工具本身也可能出错，不能把解码出的“Latent Thought”直接当作真实机制。

未来比较模型时，除了准确率，还需要记录可见 Token、隐藏步骤、FLOPs、延迟、可解释性和可审计性。Tokenization 决定模型首先看到什么，Latent Representation 影响模型怎样表示和计算，输出解码决定模型最终说出什么；三者共同塑造模型行为。
