---
title: 模型架构：Transformer 编码器、解码器与模型分支
source: https://www.bilibili.com/video/BV1k6yWBEEmH
author: 隔壁的程序员老王
published: 2025-10-30
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型架构
  - Transformer
  - Encoder
  - Decoder
  - 资料摘要
---

# 模型架构：Transformer 编码器、解码器与模型分支

原始资料：[[raw/sources/模型原理/模型架构/Transformer如何成为AI模型的地基|Transformer如何成为AI模型的地基]]

## 核心结论

原始 Transformer 以 Encoder—Decoder 结构处理翻译：Encoder 把完整输入转换为内部表示，Decoder 结合这组表示和已经生成的目标 Token，自回归地产生下一个 Token。后续模型根据任务分化为 Decoder-only 与 Encoder-only 等路线；它们共享 Transformer 基础模块，但训练目标、可见上下文和输出方式不同。

## 三种架构路线

| 架构 | 输入与输出方式 | 资料中的训练示例 | 典型定位 |
| --- | --- | --- | --- |
| Encoder—Decoder | Encoder 处理完整原文，Decoder 逐 Token 生成译文 | 成对原文与译文的监督学习 | 翻译等序列到序列任务 |
| Decoder-only | 根据已有 Token 持续预测下一个 Token | 从普通文本自动构造下一个 Token 目标 | GPT 式自回归生成 |
| Encoder-only | 同时利用被遮挡位置两侧的上下文 | BERT 的 `[MASK]` 恢复 | 文本理解、实体提取和标注 |

## 生成与采样

Decoder 每一步都为词表中的 Token 计算分数或概率，再通过解码策略选择下一项。Temperature 调整采样随机性；Top-K 只保留概率最高的固定数量 Token。原始资料已校正字幕中的 `top-p` 转写：`K=1`、`K=2` 描述的是按固定数量保留 Token 的 Top-K，Top-P 则按累计概率阈值确定保留集合。

资料借 GPT-2 的 50257 个 Token 说明输出层规模，并用开始标记—逐 Token 生成—结束标记展示自回归过程。该示例用于解释流程，不代表原始 Transformer 翻译模型与 GPT-2 使用相同词表。

## 证据边界

- “含义矩阵”是对 Encoder 内部表示的教学类比，不表示该矩阵能够被人直接翻译成稳定、唯一的语义。
- GPT-4 的 1.8 万亿参数属于资料明确标注的外界传言，没有得到 OpenAI 正式确认，不能作为已公布规格。
- Decoder 的逐 Token 特性可以解释输出阶段的串行计算压力，但 API 输入、输出价格还受到模型、硬件、缓存、并发和服务策略影响，不能只由本例推出固定价差。
- 资料对 GPT、Claude、Gemini、DeepSeek 与 Kimi 的 Decoder-only 归类是宏观概括，不能代替各模型版本的官方架构说明。
- 字幕中的“标注参考线”无法从官方字幕或章节精确恢复，正式文章只保留能够确认的“其他文本标注任务”，未补写具体任务名。

## 系列定位与关联

本资料是官方“从0开始一起学大模型”合集首条，内容建立 Transformer 宏观结构，并在结尾预告继续拆解内部组件；因此定位为系列导论，不另行添加期数。

- 下一条：[[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- 注意力内部机制：[[wiki/sources/模型架构：多头注意力与 QKV]]
- Token 与隐藏表示：[[wiki/sources/模型原理：Token Space 与 Latent Space]]
- Prefill、Decode 与 API 成本：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
