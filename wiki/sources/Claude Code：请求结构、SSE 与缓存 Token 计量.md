---
title: Claude Code：请求结构、SSE 与缓存 Token 计量
source: https://www.bilibili.com/video/BV1G2o5BqELx
author: 张司机在路上
published: 2026-04-24
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - SSE
  - Prompt Caching
  - 上下文工程
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：请求结构、SSE 与缓存 Token 计量

原始资料：[[raw/sources/应用工程/驾驭工程/解密Claude Code和Anthropic后端是如何通信|解密Claude Code和Anthropic后端是如何通信]]

## 核心结论

一次只输入 `hello` 的 Claude Code 抓包中，三个输入计量项合计 30,986 Token，最终输出 15 Token。主要输入不是五个可见字母，而是 Hook、延迟工具清单、MCP 指南、Skill、`CLAUDE.md`、系统规则、环境信息、工具 Schema 与缓存内容。评估 Coding Agent 的上下文和费用时，必须观察完整 API 请求。

## 请求结构

- `messages` 只有一条 `user` 消息，但包含六个 Content Block：前五块是 `<system-reminder>` 注入，依次承载 SessionStart Hook 附加上下文、延迟工具清单、MCP 指南、Skill 列表和项目 `CLAUDE.md`；最后一块才是 `hello`。
- `hello` Block 标记 `cache_control.type: ephemeral` 和 `ttl: 1h`。资料将它解释为缓存切割点，不能据此认定所有 Claude Code 版本都采用相同切分。
- `system` 由四个 Text Block 组成，分别承载计费头、Claude Code 身份、核心行为规则，以及输出、自动记忆和环境说明。
- 本次 `tools` 字段包含 10 个完整工具定义。延迟工具清单与这里的已加载工具不是同一对象：前者只提供名称并等待 `ToolSearch`，后者直接占用请求上下文。

## SSE 响应

响应通过 SSE 依次发送 `message_start`、`content_block_start`、`ping`、两个 `content_block_delta`、`content_block_stop`、`message_delta` 和 `message_stop`。文本由 Delta 逐块推送；`message_delta` 的 `stop_reason` 为 `end_turn`，表示模型主动结束回复。

## Token 计量

抓包中的输入项为：

$$
6\ \text{input}+14{,}145\ \text{cache creation}+16{,}835\ \text{cache read}=30{,}986
$$

`message_start` 初始记录 4 个输出 Token，`message_delta` 最终更新为 15，因此合计为 31,001 Token。资料称缓存读取比普通输入便宜九成；这是视频发布时的价格口径，不是跨模型、跨版本的固定费用。

## 证据边界

- 抓包使用 `claude-opus-4-7`，并包含作者个人安装的 Superpowers Hook、Context7 MCP、Skill 和 `CLAUDE.md`；这些内容不代表其他用户或版本的固定请求。
- 画面中的 `thinking.type` 为 `adaptive`、`output_config.effort` 为 `high`；旁白却把 `xhigh` 称为默认档位，两处冲突，因此只采用 JSON 对本次请求的记录。
- `claude-trace` 与 `cchistory` 用于观察请求和版本变化；抓包结果说明客户端发送了什么，不能单独证明 Anthropic 服务端的内部实现。

## 关联

- 下一篇：[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本]]
- WebSearch 子 Agent：[[wiki/sources/Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离]]
- ToolSearch 延迟工具：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- Codex 请求对照：[[wiki/sources/Codex：请求结构、服务端通信与 Token 计量]]
- Claude Code 运行时：[[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
