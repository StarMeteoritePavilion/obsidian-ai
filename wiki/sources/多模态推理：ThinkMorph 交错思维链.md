---
title: 多模态推理：ThinkMorph 交错思维链
source: https://www.bilibili.com/video/BV1SFCqBFEfC
author: 唐国梁Tommy
published: 2025-11-17
ingested: 2026-09-03
updated: 2026-09-03
tags:
  - AI
  - 模型原理
  - 多模态模型
  - ThinkMorph
  - Interleaved-CoT
  - 资料摘要
---

# 多模态推理：ThinkMorph 交错思维链

原始资料：[[raw/sources/模型原理/多模态模型/ThinkMorph：多模态交错思维链的三种涌现能力|ThinkMorph：多模态交错思维链的三种涌现能力]]

## 核心结论

ThinkMorph 以 BAGEL-7B 为基座，使用 24,990 条高质量轨迹进行微调，使统一模型在文本规划、图像生成或操作、视觉验证和文本回答之间交替推进。资料认为文本与图像是互补模态：前者负责抽象规划与逻辑计算，后者负责精确定位和视觉验证。

## 三种涌现能力

模型会生成训练轨迹中没有出现过的 Multiple bboxes、Motion Forecasting、Perspective Shift 和 Crop 等视觉操作；也会依据视觉信息充分性及图像生成成本，在交错推理与纯文本推理间自主切换。Best-of-N 实验中，交错推理随采样数增加保持更强扩展性，作者将其归因于文本与视觉两个解空间带来的路径多样性。

## 数据与结果边界

训练集包括 Jigsaw Assembly 6,000 条、Spatial Navigation 6,000 条、Visual Search 6,990 条和 Chart Refocus 6,000 条。资料报告视觉中心任务相对基座平均提高 34.74%，VSP 从 0.83% 提高到 86.67%，域外 SAT 为 52.67%，MMVP 为 80.33%。这些数字属于论文的特定基座、数据、基准和评测流程。

模式切换数据存在来源内部冲突：讲解页文字称 MMVP 上有 5.3% 的案例切换为纯文本，准确率由交错推理的 73.96% 变为 81.25%；同页引用的论文 Figure 5 却把 5.3% Cases 标为 Chart Refocus，并另列 Jigsaw Assembly 9.3% Cases。本库不合并这两种归属。

## 局限性

样本规模约为 2.4 万，任务类型集中；图像 Token 带来较高推理成本；结果依赖 BAGEL-7B；部分任务由 GPT-5 评判，可能受评判模型偏好、知识边界和错误影响。交错推理因此不是所有多模态问题的默认路径，视觉步骤不能增加信息时，纯文本推理可能更准确且更便宜。

## 关联

- 推理表示综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- Token 与内部连续表示：[[wiki/sources/模型原理：Token Space 与 Latent Space]]
