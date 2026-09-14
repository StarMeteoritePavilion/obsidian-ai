---
title: 揭秘Codex是如何帮你省token的
source: https://www.bilibili.com/video/BV17wN16MEUG
author: 张司机在路上
created: 2026-07-13
tags:
  - AI
  - Codex
  - OpenAI
  - Prompt Caching
  - KV Cache
  - 推理优化
  - 模型工程
---

# 揭秘Codex是如何帮你省token的

提示词缓存是 Coding Agent 控制重复计算与输入成本的重要机制。Claude Code 团队工程师 Thariq 曾撰写 *Prompt Cache is Everything*，强调缓存优化在 Agent 工作负载中的重要性。Codex 会自动管理提示词缓存；要理解它怎样复用多轮对话，首先需要观察 `usage` 中的 `input_tokens` 与 `cached_tokens`。

## 两组缓存命中实验

作者先在同一段 Codex 对话中依次发送 `Hello`、`Fine` 和 `Thank you`。三轮抓包结果如下：

- 第一轮：`input_tokens = 22852`，`cached_tokens = 4480`。
- 第二轮：`input_tokens = 22878`，`cached_tokens = 22400`。
- 第三轮：`input_tokens = 22901`，`cached_tokens = 22400`。

第二轮输入比第一轮只增加了少量对话内容，缓存命中却从 4,480 跳到 22,400；第三轮输入继续增加，缓存命中仍停在 22,400。以第二轮为例，22,878 个输入 Token 中只有 478 个没有命中缓存。

作者随后在每轮发送一段 300 多字的英文文本。进入稳定命中后，`cached_tokens` 依次为 `22400`、`22912`、`23424` 和 `23936`，每次增加 512。这个抓包现象说明，所测版本报告的缓存命中数并没有随着每个新增 Token 连续增长，而是跨过特定边界后再向上跳。

## Codex 请求中的稳定前缀

根据 OpenAI 工程文章 *Unrolling the Codex agent loop*、开发者教程 *Prompt Caching 201* 和作者对抓包结构的整理，Codex 请求可以按顺序理解为五段：

1. `instructions`：基础系统提示词。
2. `tools`：工具定义。
3. `input` 中 `role = developer` 的运行时注入，例如权限与审批规则。
4. `input` 中 `role = user` 的项目注入，例如 `AGENTS.md` 与 `environment_context`。
5. 用户对话。

系统规则、工具定义、运行时配置和项目上下文位于对话之前。只要这些内容及其顺序保持稳定，多轮请求就能保留一段较长的相同前缀；新消息追加在末尾，不会改写已经稳定的前部。

作者将这一顺序与 Claude Code 作了对比：其观察到的 Claude Code 请求先放工具定义，再放系统提示词和用户消息。作者认为 Codex 把更稳定的系统提示词放在工具定义之前更合理。这是作者对两种请求结构的比较与判断，不代表所有版本都保持相同顺序。

## 用 Automatic Prefix Caching 理解块级复用

OpenAI 当时公开的资料没有充分说明内部缓存实现，因此作者借用 vLLM 的 Automatic Prefix Caching 解释抓包现象。两者表现相似不等于 OpenAI 已确认采用同一实现，下面的 Block、哈希表和匹配过程属于这一类比。

Automatic Prefix Caching 先把提示词切成固定大小的 Block。为便于说明，假设每个 Block 包含 4 个 Token：

- Block 0：`里面／个个／都是／人才`
- Block 1：`，／说话／又／好听`
- Block 2：`，／我／超／喜欢`

每个 Block 都会计算哈希。当前 Block 的哈希不只依赖自身 Token，还依赖前一个 Block 的哈希：

`H_n = hash(H_{n-1}, tokens_n)`

Block 0 前面没有其他 Block，因此从空哈希开始。这样的链式关系带来一个性质：如果某个 Block 的哈希一致，那么从请求开头到该 Block 的前缀也一致。系统可以用哈希快速判断能够复用到哪个位置。

## 缓存保存的是 KV Cache

在这个类比中，缓存表现为一张哈希表：键是每个 Block 的哈希，值是该 Block 中所有 Token 已经计算出的 KV Cache。模型完成 Prefill 后，不丢弃这些 Key、Value 状态，而是按 Block 写入缓存。

新请求到来后，系统重新切块、计算哈希并从前向后查表。假设新请求只把最后四个 Token 从`，／我／超／喜欢`改成`，／我／不想／走`：

- Block 0 的 Token 和前序状态都没有变化，哈希命中，直接复用对应 KV Cache。
- Block 1 同样命中，继续复用对应 KV Cache。
- Block 2 的 Token 已经变化，哈希不再匹配，需要重新执行 Prefill。

前两个 Block 共 8 个 Token 被计入缓存命中，第三个 Block 整块重算。Block 要么完整命中，要么完整重算，不存在命中半个 Block 的情况。

这一机制类比可以解释实验中 `cached_tokens` 每次增加 512 的现象：当可复用前缀跨过下一个报告边界，缓存命中数才继续增加。不过，实验只能确认所测 Codex 与模型版本呈现了 512 Token 的递增结果，不能单凭该结果证明 OpenAI 内部物理 Block 的固定大小就是 512，也不能证明其实现与 vLLM 完全相同。
