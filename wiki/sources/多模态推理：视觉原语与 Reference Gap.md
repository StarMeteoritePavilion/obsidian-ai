---
title: 多模态推理：视觉原语与 Reference Gap
source: https://www.bilibili.com/video/BV15j5u6WE5C
author: 唐国梁Tommy
published: 2026-05-12
ingested: 2026-09-03
updated: 2026-09-05
tags:
  - 视觉与多模态
  - AI
  - 模型原理
  - 多模态模型
  - 多模态推理
  - 视觉原语
  - 资料摘要
---

# 多模态推理：视觉原语与 Reference Gap

原始资料：[[raw/sources/模型原理/视觉与多模态/多模态模型的真正瓶颈不是“看不清”，而是“指不准”？｜DeepSeek 视觉原语论文从架构到 7056× 压缩链路全解析|DeepSeek 视觉原语与 7056× 压缩链路]]

## 核心结论

《Thinking with Visual Primitives》区分了 Perception Gap 与 Reference Gap：提高分辨率和视觉 Token 可以缓解“看不清”，却不能保证模型在长推理链中稳定引用已经看见的对象。该方法把点和边界框直接写入中间推理，使视觉坐标成为可重复引用的“最小思维单元”。

## 表示与架构

边界框适合具备明确实体边界的定位、计数和文字查找，点序列适合迷宫、曲线与运动轨迹；坐标统一归一化到 0～999。系统采用类 LLaVA 架构，以 DeepSeek-ViT 编码图像，并把视觉与文本 Token 输入 DeepSeek-V4-Flash `284B-A13B` MoE 模型。

视觉效率来自两层压缩：ViT 将相邻 $3\times3$ Patch Token 合并，CSA 再把每 4 个视觉 Token 压缩为 1 个 KV Cache 条目。$756\times756$ 图像由 571,536 个像素依次变为 2,916 个 Patch Token、324 个语言模型视觉 Token 和 81 个 KV Cache 条目；7,056× 是像素数与缓存条目数之比，不是信息论无损压缩率。

## 结果与边界

对于 $800\times800$ 图像，资料给出的视觉 KV Cache 条目约为本文模型 90、Claude Sonnet 4.6 为 870、Gemini 3 Flash 为 1,100。论文选取的七项计数与空间推理评测平均分中，本文模型为 77.2，Gemini 3 Flash 为 76.5；这些评测只覆盖论文关注的部分维度，不能代表整体模型能力。

本资料是画面明确标注的第十期上集。本集没有讲解数据构造、专家训练、奖励设计和迷宫或路径实验，这些内容已由下集 [[wiki/sources/多模态推理：视觉原语的数据、训练与奖励|视觉原语的数据、训练与奖励]] 补充。

## 关联

- 多模态技术地图：[[wiki/sources/多模态模型：架构、数据、推理与检索]]
- 交错图文推理：[[wiki/sources/多模态推理：ThinkMorph 交错思维链]]
- 推理表示综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- Token 与内部表示：[[wiki/sources/模型原理：Token Space 与 Latent Space]]
- 下集：[[wiki/sources/多模态推理：视觉原语的数据、训练与奖励]]
