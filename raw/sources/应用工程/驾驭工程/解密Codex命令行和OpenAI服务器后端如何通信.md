---
title: 解密Codex命令行和OpenAI服务器后端如何通信
source: https://www.bilibili.com/video/BV1VJ7j6jE4L
author: 张司机在路上
created: 2026-06-26
tags:
  - AI
  - Codex
  - OpenAI
  - Responses API
  - 上下文工程
  - 驾驭工程
  - 应用工程
---

# 解密Codex命令行和OpenAI服务器后端如何通信

一次只有 `hello` 的 Codex 请求，模型仅回复“Hello. What would you like to work on?”，但抓包记录的总用量达到 14,708 Token。其中输入为 14,694 Token，缓存命中 2,432 Token，输出只有 14 Token。真正占据上下文的不是 `hello`，而是 Codex 在请求中同时携带的系统规则、运行时配置、项目上下文和工具定义。

## 从抓包记录观察 Coding Agent

作者使用 `claude-tap` 作为本地代理和轨迹查看器，记录 Codex 与服务端之间的请求，并将系统提示词、对话历史、工具 schema、工具调用、流式响应、Token 用量和请求差异生成为 HTML 页面。资料同时介绍了基于 `claude-tap` 数据构建的 `phistory.cc`，用于对比多种 Coding Agent 在不同版本中的系统提示词变化。

`phistory.cc` 延续了 `cchistory` 的版本对比思路。`cchistory` 由 Pi Agent 开发者 Mario Zechner 搭建，资料显示其 Claude Code 记录更新至 2.1.112；`phistory.cc` 则继续追踪后续变化，并将对比范围扩展到 Codex CLI 等其他 Agent。

## 请求中的四块上下文

从 JSON 外层看，主体是 `instructions`、`input` 和 `tools` 三部分；按实际内容拆分，`input` 又包含 `developer` 与 `user` 两类消息，因此可以归纳为四块上下文。

### `instructions`：基础系统规则

`instructions` 定义 Codex 的基础行为，包括身份、沟通方式、任务推进、代码修改与完成汇报等规则。按资料的对照，它对应 Claude Code 请求中的 `system` 内容。

### `developer`：运行时注入

`input` 中 `role` 为 `developer` 的消息继续补充系统规则。抓包示例包含四组内容：

- `permissions_instructions` 说明 Sandbox 模式、文件读写范围、网络访问和权限提升规则。
- `collaboration_mode` 说明 Default 与 Plan 等协作模式。
- `skills_instructions` 列出当前可用 Skill 的名称、描述和来源位置。
- `plugins_instructions` 说明由 Skill、MCP 和 App 组成的本地插件集合。

### `user`：项目上下文与真实问题

`role` 为 `user` 的消息不只包含当前输入。示例先放入 `AGENTS.md` 中的项目规则，再放入 `environment_context` 所记录的当前工作目录、Shell、日期、时区和文件系统权限，最后才是用户的 `hello`。对话继续后，历史记录也会进入 `input` 数组。资料因此将整个 `input` 数组与 Claude Code 的 `messages` 进行类比。

### `tools`：可调用能力的结构化定义

`tools` 列出模型生成回复时可以调用的工具及其 schema。资料将 `shell_command` 类比为 Claude Code 的 Bash，将 `request_user_input` 类比为 AskUserQuestion，将 `apply_patch` 类比为 Edit 与 Write，并将 `view_image` 类比为图像读取能力。

抓包页汇总与旁白均称该次请求暴露了 16 个工具，但视频的结构示意图标注为“14 个工具定义”。两处口径不一致，因此这些画面只能支持“工具定义会占用输入上下文”，不能将具体数量当作 Codex 的固定工具数。

## 推理、留存与缓存参数

请求体尾部还有三项值得关注的参数：

- `reasoning.effort` 为 `high`，表示该轮使用高推理强度。
- `store` 为 `false`，资料将其解释为服务端不留存对话状态，由当前使用 HTTPS 的 Codex 客户端在每次请求中发送完整记录。资料同时称，使用 WebSocket 时该值会变为 `true`，并只发送新增上下文；这是作者对所抓版本的观察，不是所有 Codex 版本的固定协议。
- `prompt_cache_key` 用于 OpenAI 提示词缓存。该视频只标出参数及其用途，把具体机制留到后续视频。

## 响应的内容与 Token 用量

响应中的 `output` 数组保存模型本轮产出。普通文本回复表现为 `assistant` 的 `message`；模型决定调用工具时，同一数组也可以出现 `function_call` 等调用项。

`usage` 记录输入、缓存命中、输出和总 Token 数。本次抓包中，真实问题只有 1 Token，`instructions`、`developer` 和项目上下文、`tools` 合计约 14,693 Token，模型回复占 14 Token，最终总计 14,708 Token。这个单次抓包说明，评估 Agent 请求成本时不能只数用户输入，还必须统计系统规则、运行时注入、项目上下文、历史和工具定义。

## 两种缓存产品取向

资料将 Claude Opus 4.8 与 GPT-5.5 发布时的标准价格放在一起对比。Claude Opus 4.8 每百万 Token 的基础输入、一小时缓存写入、缓存命中和输出价格分别为 5、10、0.5 和 25 美元。GPT-5.5 在短上下文档位的输入、缓存输入和输出价格分别为 5、0.5 和 30 美元；长上下文档位分别为 10、1 和 45 美元。画面以 272K Token 区分短上下文与长上下文。

OpenAI 的 `usage` 没有单列缓存写入，Anthropic 则对缓存写入与命中分别定价。作者据此将 Claude Code 视为更偏向开发者自主控制 `cache_control` 的 To B 产品，将 Codex 视为自动管理缓存、定价更容易理解的 To C 产品。这是作者对当时产品定位与价格结构的解释，不是两类用户或缓存实现的通用分类。
