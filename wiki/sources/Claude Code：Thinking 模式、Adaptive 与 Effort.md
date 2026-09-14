---
title: Claude Code：Thinking 模式、Adaptive 与 Effort
source: https://www.bilibili.com/video/BV1fc5j62E1Q
author: 张司机在路上
published: 2026-05-10
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - Thinking
  - Reasoning
  - Transformer
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：Thinking 模式、Adaptive 与 Effort

原始资料：[[raw/sources/应用工程/驾驭工程/Thinking模式是如何让Claude Code变聪明的|Thinking模式是如何让Claude Code变聪明的]]

## 核心结论

资料将普通模式概括为“上下文 → 行动”，将 Thinking 模式概括为“上下文 → 中间推理 → 行动”。当次 Claude Code 抓包的请求使用 `thinking: adaptive` 与 `effort: high`；响应中的 Thinking Block 先定位 TTL 缓存的 `get` 没有检查过期时间，Text Block 再给出 `Date.now()`、`entry.expiry`、删除条目并返回 `null` 的 Lazy Eviction 修复方案。

## 运行时控制

- `thinking: adaptive` 让模型根据任务复杂度决定是否思考及思考深度，不表示每次请求都强制生成相同长度的推理。
- `effort: high` 表示当次请求的推理强度。资料列出的完整档位为 `low`、`medium`、`high`、`xhigh` 和 `max`；提高档位会增加可用思考 Token，同时带来更高延迟与 Token 消耗。
- Thinking 模式在 `content` 数组中把 `type: thinking` 的 Block 放在 Text Block 之前。自回归生成时，先前生成的 Thinking Token 会成为后续 Text Token 的上下文。

## TTL 缓存案例

`cache.js` 提供 `set` 与 `get`，`app.js` 把 `user-1` 写入缓存并设置一秒 TTL，两秒后读取时预期为 `null`，实际仍返回原值。Thinking Block 将根因定位为 `get` 只读取值、没有检查过期时间；Text Block 建议比较 `Date.now()` 与 `entry.expiry`，过期后从 `Map` 删除并返回 `null`。资料把这种读取时清理称为 Lazy Eviction。

该案例展示了中间推理与正式答案的内容承接，但只观察到一次问题和一次响应，没有提供关闭 Thinking 后的同题对照、准确率统计或延迟与 Token 数值，因此不能从本案例量化 Thinking 模式的普遍收益。

## 训练解释与证据边界

资料把训练过程概括为预训练、指令微调、RLHF 与 Reasoning Tuning，并用 $2(x-3)=14$ 的逐步样本解释模型怎样学习“先推理、后回答”的输出习惯。这是资料对推理能力来源的教学性说明，没有给出对应 Claude 模型的训练数据、训练配方或消融实验，不能据此认定所有 Sonnet、Opus 或其他推理模型都遵循相同的四阶段流程。

可见 Thinking Block 是 Token Space 中生成的中间文本，不等于模型全部内部计算，也不自动保证推理过程忠实或结论正确。运行时字段、训练所得能力和最终任务质量属于三个需要分别验证的层次。

## 关联

- 合集上一篇：[[wiki/sources/Claude Code：cache_control 断点与 20 Block 前缀回溯]]
- 请求结构：[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- 合集下一条：[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- 可见推理与内部计算：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- 后训练：[[wiki/syntheses/大模型后训练：从模仿到行为选择]]
- 运行时控制：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
