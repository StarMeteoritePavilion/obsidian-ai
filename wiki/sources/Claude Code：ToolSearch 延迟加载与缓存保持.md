---
title: Claude Code：ToolSearch 延迟加载与缓存保持
source: https://www.bilibili.com/video/BV1Bd7X64EeF
author: 张司机在路上
published: 2026-06-03
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - ToolSearch
  - Deferred Tools
  - Prompt Caching
  - MCP
  - 上下文工程
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：ToolSearch 延迟加载与缓存保持

原始资料：[[raw/sources/应用工程/驾驭工程/Claude Code如何利用ToolSearch调用任意多个工具|Claude Code如何利用ToolSearch调用任意多个工具]]

## 核心结论

ToolSearch 将工具发现与工具使用拆成两轮：System Prompt 先列出 Deferred Tool 名称，模型需要某项能力时用 `select:<tool_name>` 请求完整定义；下一轮再通过带 `defer_loading: true` 的工具定义和 `tool_reference` 让模型正式调用。大量工具因此不必全部预载。

## 调用链路

抓包开始时，`tools` 数组含 10 个完整定义，System Prompt 另列 23 个只有名称的 Deferred Tools。示例中，模型先调用：

```json
{
  "query": "select:AskUserQuestion",
  "max_results": 1
}
```

下一轮 `tools` 增加完整的 `AskUserQuestion`，对话末尾增加 `{"type":"tool_reference","tool_name":"AskUserQuestion"}`。模型得到 Schema 后才发起真正的询问工具调用。ToolSearch 因而多一次往返，但只为当前需要的工具支付完整定义成本。

## 缓存保持

资料称，带 `defer_loading: true` 的工具不参与原有 Cache 前缀 Hash，`tool_reference` 则追加在历史末尾并由服务端展开。新增工具不会改变此前的 `tools → system → messages` 前缀，Prompt Cache 因而可以继续命中。

`ANTHROPIC_BASE_URL` 指向非 Anthropic 代理时，Claude Code 默认关闭 Tool Search，因为多数代理不转发 `tool_reference`。`ENABLE_TOOL_SEARCH=true` 可以强制开启，但官方配置表明确提示，不支持相关协议的代理可能请求失败。

## 数据与边界

Anthropic 的五服务器示例包含 GitHub、Slack、Sentry、Grafana 和 Splunk，共 58 个工具、约 55K Token。传统方式为 50 多个工具定义约 72K Token，工作开始前总上下文约 77K；Tool Search 自身约 500 Token，按需加载 3～5 个工具约 3K，总上下文约 8.7K，文章称 Token 用量降低 85%。

同一 MCP 评估中，Opus 4 的工具选择准确率从 49% 提升到 74%，Opus 4.5 从 79.5% 提升到 88.1%。这些数字来自对应文章的配置，不能外推为其他模型和工具库的固定收益。

本地字幕把五个服务写成含 Jira、GitLab 的组合，把 8.7K 写成 8K，并把 Opus 4／4.5 写成 Opus 3／Sonnet 3.5；正式资料依据视频展示的 Anthropic 原文画面校正。

## 关联

- 下一篇：[[wiki/sources/Claude Code：compact 上下文压缩与工作现场恢复|Claude Code：/compact 上下文压缩与工作现场恢复]]
- Skill 渐进式披露：[[wiki/sources/Claude Code：Skill 渐进式披露与第三方执行边界]]
- Claude Code 请求结构：[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- 启动配置与缓存失效：[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- WebSearch 服务器端执行：[[wiki/sources/Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离]]
- Codex 工具与请求对照：[[wiki/sources/Codex：请求结构、服务端通信与 Token 计量]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
