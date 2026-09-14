---
title: 模型推理优化：Codex 自动前缀缓存
source: https://www.bilibili.com/video/BV17wN16MEUG
author: 张司机在路上
published: 2026-07-13
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Codex
  - OpenAI
  - Prompt Caching
  - KV Cache
  - 推理优化
  - 模型工程
  - 资料摘要
---

# 模型推理优化：Codex 自动前缀缓存

原始资料：[[raw/sources/模型工程/推理与服务/揭秘Codex是如何帮你省token的|揭秘Codex是如何帮你省token的]]

## 核心结论

Codex 的提示词缓存依赖请求开头的稳定前缀。资料所测三轮短对话中，输入 Token 从 22,852 增至 22,878、22,901，缓存命中从 4,480 跳至 22,400 后保持不变；第二轮只有 478 个输入 Token 未命中。长文本实验中的 `cached_tokens` 则为 `22400 → 22912 → 23424 → 23936`，呈现 512 Token 的递增现象。

稳定前缀按顺序包含 `instructions`、`tools`、`input` 中的 `developer` 注入、`user` 项目上下文和用户对话。系统规则、工具定义、权限配置、`AGENTS.md` 与 `environment_context` 位于对话前部，内容及顺序稳定时更容易跨轮复用；较早位置的变化会破坏其后的前缀匹配。

## 机制类比

资料明确指出 OpenAI 当时没有公开足够的内部实现细节，因此借用 vLLM Automatic Prefix Caching 作类比：提示词按固定大小切成 Block；当前 Block 的哈希依赖上一 Block 哈希与自身 Token；哈希表以 Block 哈希为键、对应 KV Cache 为值；新请求从前向后查找，命中的 Block 直接复用 KV Cache，首个不匹配位置及其后续前缀重新执行 Prefill。

这个类比能解释“整块命中或整块重算”，但不能证明 OpenAI 的物理缓存 Block 固定为 512 Token，也不能证明 OpenAI 与 vLLM 使用相同实现。

## 当前官方文档边界

截至 2026-09-14，OpenAI [Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching) 文档说明：缓存保存的是 KV Tensor，而不是 Token 本身；完整渲染上下文包含 OpenAI 指令、Developer Message、工具定义和对话历史；复用要求完整前缀匹配。该文档还说明 GPT-5.5 及更早模型的 `cached_tokens` 会向下取整为 128 的倍数。

因此，视频观察到的 512 递增与当前文档所述 128 报告粒度处于不同版本与解释层级。512 是该次抓包的观测间隔，不足以推出物理 Block 大小；128 是当前文档对 `cached_tokens` 报告取整的说明，也不表示每次命中一定只增加 128。

## 关联

- [[wiki/sources/模型推理优化：KV Cache 与 Prompt Cache 的复用层级]]
- [[wiki/sources/模型推理优化：PagedAttention 分页、前缀共享与驱逐]]
- [[wiki/sources/Codex：请求结构、服务端通信与 Token 计量]]
- [[wiki/sources/大语言模型：Tokenizer、Token ID 与 BPE]]
- [[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- [[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
