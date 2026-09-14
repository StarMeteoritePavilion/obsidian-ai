---
title: 揭秘在Claude Code里WebSearch是如何搜索网页的
source: https://www.bilibili.com/video/BV1CgRzBnEvM
author: 张司机在路上
created: 2026-05-05
tags:
  - AI
  - Claude Code
  - Anthropic
  - WebSearch
  - Server Tools
  - Subagent
  - Prompt Injection
  - Brave
  - 上下文工程
  - 驾驭工程
  - 应用工程
---

# 揭秘在Claude Code里WebSearch是如何搜索网页的

大模型本身不能直接操作外部环境，需要通过工具完成任务。工具可以按执行位置分为两类：`Bash` 等客户端工具运行在用户电脑上，WebSearch 等服务器端工具则由 Anthropic 后端执行。

作者让 Claude Code 查询 2026 年 4 月 30 日的伦敦天气，并用 `claude-tap` 抓取完整请求。WebSearch 的工具说明写着：

> Searches are performed automatically within a single API call.

按照这段说明，搜索似乎会在一次 API 调用中自动完成。然而，这次 Claude Code 端到端执行实际包含三次 API 请求，并在 Opus 4.7 与 Haiku 4.5 两个模型之间切换。

## 第一次请求：Opus 决定是否搜索

第一次请求由主 Agent 使用 Opus 4.7 处理。`messages` 包含用户对伦敦天气的提问，`tools` 数组包含 28 个工具。

Opus 没有直接返回搜索结果，而是输出一个 `tool_use` Block：工具名为 `WebSearch`，参数包含伦敦天气和当天日期，`stop_reason` 为 `tool_use`。这种返回格式与客户端工具相同，表示模型已经决定搜索，并等待 Claude Code 客户端继续处理。

因此，工具描述所说的“一次 API 调用”没有等同于这次 Claude Code 完整任务只发出一次请求。主 Agent 的第一次调用只负责决策，真正的服务器端搜索发生在下一次请求中。

## 第二次请求：Haiku 执行服务器端搜索

Claude Code 收到搜索指令后，另开一个对话，改用 Haiku 4.5 执行。这个子 Agent 的工具列表只有 `web_search` 一项，其类型为 `web_search_20250305`。系统提示词也被压缩为直接的任务说明：

> You are an assistant for performing a web search tool use

用户消息则直接要求搜索：

> Perform a web search for the query: London weather today April 30 2026

响应首先返回 `server_tool_use`。其 ID 以 `srvtoolu_` 开头，工具名为 `web_search`，表明这次调用确实进入服务器端执行。紧随其后的是 `web_search_tool_result`：本次抓包得到 10 条搜索结果，每条包含 URL、标题和 `encrypted_content`，加密内容合计约 30 KB。

`encrypted_content` 不是 Haiku 生成的文字，而是 Anthropic 服务器返回的加密字段。客户端可以读取 URL、标题和引用，却无法直接解读搜索正文。作者据此解释，加密层既允许模型使用搜索内容生成回答，也避免 API 客户端把服务当作网页正文下载接口；本次搜索的底层能力来自 Brave。

Haiku 在处理所有搜索结果后，还生成了一段关于伦敦天气的英文摘要。响应的 `usage` 中，`server_tool_use.web_search_requests` 为 1，用于记录本次额外的网页搜索计费。

## 第三次请求：Opus 生成最终回答

第三次请求切回 Opus 4.7。Claude Code 传给主 Agent 的 `tool_result` 包含 10 条链接的 URL 与标题，以及 Haiku 生成的天气摘要。占用最多内容的 `encrypted_content` 没有进入主 Agent 上下文。

主 Agent 因而不需要读取服务器端搜索的完整中间数据，只使用链接、标题和摘要组织面向用户的最终回答。整个 WebSearch 流程可以概括为：

1. Opus 4.7 判断是否需要搜索，并生成工具调用参数。
2. Haiku 4.5 在只有 `web_search` 的独立上下文中执行服务器端搜索并压缩结果。
3. Opus 4.7 接收链接与摘要，生成最终回复。

这次抓包因此呈现为三次 API 请求、两个模型，而不是主 Agent 在一次请求中直接取得最终答案。

## 为什么使用 Haiku 子 Agent

### 上下文隔离

Anthropic Agent SDK 的资料将子 Agent 描述为隐藏中间过程的机制：中间步骤保留在子 Agent 内，只把最终结果交回主 Agent。

这次抓包符合该结构。约 30 KB 加密搜索内容和原始结果留在 Haiku 一侧，Opus 只接收几行摘要与链接。主上下文不需要承担完整搜索材料的 Token 和注意力负担。

### 攻击面隔离

主 Agent 的 28 个工具包含 `Bash`、`Edit` 和 `Write`，能够直接操作用户电脑；Haiku 子 Agent 只有 `web_search`。网页内容由外部发布者控制，可能携带“忽略前述指令”等 Prompt Injection 文本。如果不可信内容直接进入拥有本地操作能力的主 Agent，上下文中的恶意指令可能影响其决策。

把搜索内容交给仅有 WebSearch 的子 Agent 后，即使该上下文受到提示词注入，也不能直接调用 `Bash`、`Edit` 或 `Write`。它仍可能产生误导性文字，但无法直接触及主 Agent 的本地工具箱。

这套分工可以概括为：较昂贵的 Opus 负责判断和最终表达，较便宜的 Haiku 负责处理搜索执行与不可信内容；中间数据和高风险输入被限制在更窄的工具边界内。

## WebSearch 的独立计费

本次抓包引用的 Claude API 计费说明将 WebSearch 与标准 Token 费用分开计算，价格为每 1,000 次搜索 10 美元。客户端通过 `web_search_requests` 统计搜索次数。

作者以 OpenClaw 为对照：使用其 Brave Search API 时，用户需要自行注册 Brave 账号、提供 API Key 并支付搜索服务费用。Claude Code 的 WebSearch 则由 Anthropic 维护并支付底层搜索基础设施，用户不需要单独提供 Brave API Key。额外的 10 美元搜索费用购买的不只是工具接口，也包含这层搜索服务。

这次 WebSearch 抓包展示了 Claude Code Harness 的三项职责：由主模型决定何时调用工具，由独立子 Agent 缩减上下文和攻击面，再由平台把外部搜索基础设施封装为服务器端能力。其具体模型、工具数量、请求结构和价格属于本次抓包及视频发布时的产品状态，不能视为所有 Claude Code 版本的固定实现。
