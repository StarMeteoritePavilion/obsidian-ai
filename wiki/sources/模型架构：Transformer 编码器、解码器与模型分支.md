---
title: 模型架构：Transformer 编码器、解码器与模型分支
source: https://www.bilibili.com/video/BV1k6yWBEEmH
author: 隔壁的程序员老王
published: 2025-10-30
ingested: 2026-09-04
updated: 2026-10-10
tags:
  - AI
  - 模型架构
  - Transformer
  - Encoder
  - Decoder
  - 资料摘要
---

# 模型架构：Transformer 编码器、解码器与模型分支

原始资料：[[raw/sources/模型原理/模型架构/Transformer如何成为AI模型的地基|Transformer 架构导论：Encoder、Decoder 与生成模型]]

## 核心结论

原始 Transformer 用 Encoder 编码源序列，Decoder 结合源表示与已生成目标 Token 自回归预测下一项。原论文采用 Post-LN；后续 Pre-LN 等布局分别讨论。内部上下文化表示不是可逐维翻译为人类词义的“含义矩阵”。（[[raw/sources/模型原理/模型架构/Transformer如何成为AI模型的地基#原始Encoder：编码全部源文本|原始 Encoder]]）

## 三种架构路线

| 架构 | 可见上下文 | 常用训练目标 | 输出与用途 |
| --- | --- | --- | --- |
| Encoder—Decoder | Encoder 读取源序列；Decoder 读取源表示与已有目标 | 条件序列预测 | 翻译、摘要等输入到输出转换 |
| Decoder-only | 当前及此前 Token | 下一个 Token 预测 | 自回归文本或结构化生成 |
| Encoder-only | 全部有效输入，按任务屏蔽部分 Token | 如 BERT 的遮盖 Token 恢复 | 分类、实体提取等表示任务 |

Decoder-only 通常没有原始 Decoder 的 Encoder 交叉注意力；Encoder-only 的常用任务定位也不等于绝对无法参与生成。未公开架构的商业模型不作确定归类。（[[raw/sources/模型原理/模型架构/Transformer如何成为AI模型的地基#三类Transformer的职责|模型分支]]）

训练时将正确目标序列右移，因果 Mask 防止读取未来标签，各位置可并行计算损失；推理时则依赖刚生成的 Token 逐步推进。固定源输入的一次翻译中可复用 Encoder 输出，不表示跨请求自动共享。（[[raw/sources/模型原理/模型架构/Transformer如何成为AI模型的地基#训练不等于逐次运行完整生成|训练与推理]]）

## 生成与采样

对 $T>0$，常见温度采样为 $p_i\propto e^{z_i/T}$；$T=0$ 不能直接代入公式，接口对贪心选择的约定分别核对。Top-K 保留固定数量最高分 Token，Top-P 按累计概率阈值保留集合；两者均改变解码选择，不重新训练模型。（[[raw/sources/模型原理/模型架构/Transformer如何成为AI模型的地基#解码补充：Temperature、Top-K与Top-P|采样定义]]）

## 证据边界

输出越长通常带来更多串行生成步骤；API 价差还取决于平台、模型、硬件、缓存与并发，不能仅由架构推导报价。

## 系列定位与关联

- 下一条：[[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- 注意力内部机制：[[wiki/sources/模型架构：多头注意力与 QKV]]
- Query／Key 的相对位置编码：[[wiki/sources/模型架构：RoPE 相对位置与旋转点积]]
- Token 与隐藏表示：[[wiki/sources/大语言模型：Token、Embedding 与 Latent Space]]
- Prefill、Decode 与 API 成本：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
