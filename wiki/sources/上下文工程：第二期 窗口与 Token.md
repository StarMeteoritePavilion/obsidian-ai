---
title: 上下文工程：第二期 窗口与 Token
source: https://www.bilibili.com/video/BV1r5MH6QEgD
author: 晴天AI实战
published: 2026-07-09
ingested: 2026-07-15
updated: 2026-09-05
tags:
  - 上下文与知识工程
  - AI
  - 上下文工程
  - 应用工程
  - 资料摘要
---

# 上下文工程：第二期 窗口与 Token

原始资料：[[raw/sources/应用工程/上下文与知识工程/第二期：窗口与token|第二期：窗口与 Token]]

## 核心结论

上下文窗口是输入与输出共享的有限 Token 容量。标称窗口长度不等于稳定有效长度；位置编码、KV 缓存、注意力计算和“中间遗忘”共同限制长上下文的实际质量与成本。

## 主要内容

- 输入占用越多，可用于输出的空间越少，必须显式预留输出预算。
- 长度受位置编码、随长度线性增长的 KV 缓存以及复杂度为 $O(n^2)$ 的注意力计算约束。
- Lost in the Middle 表现为首尾信息更容易被利用，中间信息更容易被忽略。
- Token 不是字符或单词的固定映射；应使用目标模型的官方工具精确计数 Token，并据此管理容量、成本与延迟。
- 原文给出的预算比例是工程示例，实际比例应按知识密集、多轮对话或长文本生成任务分别调整。

## 关联

- GQA、DSA 与 MSA：[[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]]
- 系列综述：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 上一篇：[[wiki/sources/上下文工程：第一期 从 Prompt 到 Context]]
- 下一篇：[[wiki/sources/上下文工程：第三期 原则、策略、评估]]
