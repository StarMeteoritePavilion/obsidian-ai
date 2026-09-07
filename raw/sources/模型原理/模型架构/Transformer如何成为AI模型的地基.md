---
title: Transformer 架构导论：Encoder、Decoder 与生成模型
source: https://www.bilibili.com/video/BV1k6yWBEEmH
author: 隔壁的程序员老王
created: 2025-10-30
tags:
  - AI
  - 模型架构
  - Transformer
  - Encoder
  - Decoder
updated: 2026-09-06
---

# Transformer 架构导论：Encoder、Decoder 与生成模型

Transformer最初以Encoder–Decoder架构处理机器翻译。它先编码源文本，再依据源表示与已生成目标文本预测下一个Token。理解这一原始结构，也能分清只保留编码端或自回归生成端的模型为什么使用不同的可见范围和训练目标。

## 原始Encoder：编码全部源文本

2017年的《Attention Is All You Need》第3节描述了编码器与解码器。以把“I'm 王”译为“我是王”为例，源Token先经过嵌入并加入位置信息，再送入多层Encoder。

原始Encoder每层按顺序执行：多头双向自注意力 → 残差相加与LayerNorm → 逐位置FFN → 残差相加与LayerNorm。这里是论文原始的Post-LN布局；其他Transformer使用Pre-LN等变体，不能混成同一个实现。相同结构重复多层，并不意味着各层共享参数。

自注意力让每个源位置读取全部有效输入，FFN独立处理每个位置的向量。输出可看作上下文化表示序列，并不是一张可以逐维翻译为人类词义的“含义矩阵”。位置编码为Token顺序提供信息，否则注意力本身不会自动知道哪个词在前。

## Decoder：在目标侧屏蔽未来

原始Decoder每层包含三个子层：

1. 对已有目标Token做因果自注意力，只读取当前及更早位置。
2. 做Encoder–Decoder交叉注意力：Query来自解码端，Key/Value来自Encoder输出。
3. 用FFN处理各目标位置。

每个子层后都有残差连接与LayerNorm。最终表示经输出投影得到整个词表上的logits；logits是分数，经过归一化或采样策略后才得到概率分布与选出的Token。

例如输入起始标记后生成“我”，再以起始标记和“我”生成“是”，继续生成“王”与终止标记。这个例子只说明条件依赖，不能用人为分配的概率当成模型实测。

在**一次固定源输入的标准翻译生成**中，Encoder结果可以复用；这不意味着不同请求会自动共享同一编码结果。目标序列继续增长，自回归Decode仍要逐步推进。

## 训练不等于逐次运行完整生成

监督翻译使用成对原文、译文。训练时把正确目标序列右移一位送入Decoder，预测当前位置的正确Token；因果Mask阻止读取未来标签，因此一条样本的各个目标位置可以并行计算损失。这与推理时必须把刚生成的Token用于下一步有区别。

下一个Token预测也可以直接从普通文本构造标签。例如输入“我是老”预测“王”，目标来自文本本身，这称为自监督学习。模型参数通过损失、反向传播与优化器更新学习映射，参数规模并不能单独说明模型效果。

## 三类Transformer的职责

| 架构 | 可见上下文 | 常用训练目标 | 输出方式与用途 |
| --- | --- | --- | --- |
| Encoder-only | 全部有效输入，按任务屏蔽部分Token | 如遮盖Token恢复；BERT是这一类实例 | 输出上下文化表示，服务分类、实体提取等 |
| Decoder-only | 当前及此前Token | 下一个Token预测 | 自回归生成文本或结构化序列；GPT是这一类实例 |
| Encoder–Decoder | Encoder读取源序列；Decoder读取完整源表示与已有目标 | 条件序列预测 | 翻译、摘要等输入到输出转换 |

Decoder-only通常不包含原始Decoder中依赖独立Encoder的交叉注意力。Encoder-only虽然擅长表示学习，也不能被定义成“绝对无法用于生成”；这里比较的是常用训练和输出方式。未公开架构的商业产品不在本表中作确定归类。

## 解码补充：Temperature、Top-K与Top-P

对温度 $T>0$，常见采样分布为 $p_i\propto\exp(z_i/T)$。升高温度通常使分布更平坦，降低温度更集中。$T=0$ 不能直接代入该公式；部分API将其约定为贪心选择，具体行为应查接口文档，也不能由此保证所有硬件和并行实现逐字确定。

Top-K只保留分数最高的固定K个Token，再归一化采样；K=1时只剩一个选择。Top-P保留累计概率达到阈值的高概率集合，集合大小会随分布变化。原视频字幕把固定数量保留写成Top-P，本文按实际定义纠正为Top-K。

这些方法改变解码选择，不会重新训练模型。输出越长通常需要更多串行步骤，但价格还受平台策略、模型与服务方式影响，不能仅由架构推导真实API报价。

## 来源与关联阅读

- [Attention Is All You Need，v7](https://arxiv.org/html/1706.03762v7)：第3.1节Encoder/Decoder子层，第3.2.3节注意力输入来源，第3.3节FFN，第3.4节嵌入与输出投影，第3.5节位置信息；第5节训练设置。
- [原视频](https://www.bilibili.com/video/BV1k6yWBEEmH)：中文示例与三类结构导览。本文删除未证实的GPT-4参数传闻、商业模型统一归类、人生类比与合集预告。
- [[raw/sources/模型原理/模型架构/多头注意力 MultiHeadAttention 是什么|多头注意力]]；[[raw/sources/模型原理/基础原理/从Linear到MLP AI模型的数学本质【Transformer结构拆解】|线性层与MLP]]。
