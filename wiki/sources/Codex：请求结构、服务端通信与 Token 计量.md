---
title: Codex：请求结构、服务端通信与 Token 计量
source: https://www.bilibili.com/video/BV1VJ7j6jE4L
author: 张司机在路上
published: 2026-06-26
ingested: 2026-09-13
updated: 2026-09-14
tags:
  - AI
  - Codex
  - OpenAI
  - Responses API
  - 上下文工程
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Codex：请求结构、服务端通信与 Token 计量

原始资料：[[raw/sources/应用工程/驾驭工程/解密Codex命令行和OpenAI服务器后端如何通信|解密Codex命令行和OpenAI服务器后端如何通信]]

## 核心结论

一次只有 `hello` 的 Codex 请求仍使用了 14,708 Token：真实问题只占 1 Token，而系统规则、运行时注入、项目上下文和工具定义约占 14,693 Token，输出占 14 Token。Agent 成本必须以完整请求为单位，不能只计算用户输入。

## 请求与响应

- 请求的三个外层主体是 `instructions`、`input` 和 `tools`；将 `input` 内的 `developer` 与 `user` 分开后，可归纳为四块上下文。
- `instructions` 保存基础行为规则；`developer` 消息注入权限、协作模式、Skill 与插件清单；`user` 消息包含 `AGENTS.md`、环境上下文、对话历史和当前问题；`tools` 定义模型可调用的能力。
- `output` 既可保存普通 `message`，也可保存 `function_call` 等工具调用项；`usage` 分别记录输入、缓存命中、输出和总 Token 数。
- 资料所示 `reasoning.effort` 为 `high`、`store` 为 `false`，并带有 `prompt_cache_key`。`store` 随 HTTPS、WebSocket 而变化的表述是作者对所抓版本的观察，不能外推到所有版本。

## 证据与边界

- 本次请求为单一抓包案例，不代表其他项目、会话、模型或 Codex 版本的固定 Token 构成。
- 抓包页与旁白称共有 16 个工具，结构示意图却标注 14 个工具定义。来源内部口径不一致，因此不将具体数量写成 Codex 的固定能力数。
- Claude Opus 4.8 与 GPT-5.5 的价格为视频发布时的画面记录；其中 GPT-5.5 按 272K Token 门槛分为短、长上下文价格。这些数字具有时效性，不是当前价格保证。
- 作者将 Claude Code 归纳为允许开发者控制 `cache_control` 的 To B 产品，将 Codex 归纳为自动管理缓存的 To C 产品；这是作者的产品解释，不是通用分类。

## 关联

- [[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- [[wiki/sources/Codex：禁用 WebSocket 解决重复重连]]
- [[wiki/sources/大语言模型：Tokenizer、Token ID 与 BPE]]
- [[wiki/sources/模型推理优化：Codex 自动前缀缓存]]
- [[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
