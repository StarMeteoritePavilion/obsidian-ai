---
title: Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离
source: https://www.bilibili.com/video/BV1CgRzBnEvM
author: 张司机在路上
published: 2026-05-05
ingested: 2026-09-14
updated: 2026-09-14
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
  - 资料摘要
---

# Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离

原始资料：[[raw/sources/应用工程/驾驭工程/揭秘在Claude Code里WebSearch是如何搜索网页的|揭秘在Claude Code里WebSearch是如何搜索网页的]]

## 核心结论

一次查询伦敦天气的 Claude Code 抓包包含三次 API 请求和两个模型：Opus 4.7 先决定搜索，Haiku 4.5 在只有 `web_search` 的独立上下文中执行服务器端搜索并压缩结果，Opus 4.7 再根据 10 条链接和摘要生成最终回答。约 30 KB 的 `encrypted_content` 没有进入主 Agent，上下文与可执行工具边界因此同时收窄。

## 三次请求

1. **主 Agent 决策**：Opus 4.7 收到用户问题和 28 个工具定义，返回名为 `WebSearch` 的 `tool_use`，`stop_reason` 也为 `tool_use`。这一轮只生成搜索参数。
2. **子 Agent 搜索**：Claude Code 另开 Haiku 4.5 对话，只提供类型为 `web_search_20250305` 的 `web_search`。响应先返回 `server_tool_use`，ID 以 `srvtoolu_` 开头，再返回 `web_search_tool_result`、10 条结果、约 30 KB `encrypted_content` 和天气摘要；`server_tool_use.web_search_requests` 为 1。
3. **主 Agent 回答**：第三次请求切回 Opus 4.7。主 Agent 只接收 10 条 URL 与标题，以及 Haiku 的英文摘要，不接收搜索结果中的加密内容，再据此生成最终回复。

工具说明中的 “Searches are performed automatically within a single API call” 没有使本次 Claude Code 端到端任务缩减为一次请求。抓包显示，主 Agent 决策、服务器端搜索和最终回答分别占用一次 API 调用。

## 隔离与安全边界

资料把这项设计归纳为两层隔离：

- **上下文隔离**：搜索原始结果与加密内容留在 Haiku 子上下文，主 Agent 只获得链接和摘要，避免中间材料挤占主上下文。
- **攻击面隔离**：主 Agent 的工具包含 `Bash`、`Edit` 和 `Write`，Haiku 只有 `web_search`。网页中的 Prompt Injection 即使影响子 Agent，也不能直接调用本地文件和命令工具；剩余风险是误导性文字继续影响主 Agent。

这种结构同时涉及模型路由、上下文压缩和最小工具权限。它减少了不可信网页内容直接接触高权限工具的机会，但不等同于验证搜索事实或消除提示词注入。

## 加密内容与计费

资料称，Anthropic 的搜索能力来自 Brave。搜索结果向模型提供 URL、标题、引用和加密正文，客户端无法直接读取 `encrypted_content`。作者将这一设计解释为：允许模型基于搜索正文回答，同时避免 API 客户端把服务当作网页正文下载接口。该用途属于作者根据抓包和服务关系给出的解释，不是抓包字段本身能够独立证明的协议目的。

视频展示的 Claude API 计费说明为每 1,000 次 WebSearch 10 美元，且与标准 Token 费用分开统计。OpenClaw 对照案例需要用户自行配置并支付 Brave Search API；Claude Code 则把底层搜索基础设施封装进服务器端工具。价格与服务安排属于视频发布时口径，后续使用应重新核对当前文档。

## 证据边界

- 三次请求、Opus 4.7／Haiku 4.5、28／1 个工具、10 条结果和约 30 KB 加密内容均来自作者这一次 Claude Code 配置与抓包，不能外推为所有版本的固定结构。
- `server_tool_use`、`srvtoolu_`、`web_search_tool_result`、`encrypted_content` 和 `web_search_requests` 可由抓包画面确认；抓包只能观察客户端请求与服务端返回，不能直接观察 Anthropic 内部执行过程。
- 独立子 Agent 缩小了直接可用工具集合，却仍可能输出误导文字；隔离降低攻击面，不构成搜索结果可信性证明。
- 每千次 10 美元和 Brave 服务关系属于视频展示及发布说明的时间点，不能视为长期不变的计费与供应商承诺。

## 关联

- 上一篇：[[wiki/sources/Claude Code：tool_use、tool_result 与客户端工具闭环]]
- 下一篇：[[wiki/sources/Claude Code：cache_control 断点与 20 Block 前缀回溯]]
- 请求结构：[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- 多轮缓存：[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本]]
- 缓存命中边界：[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- 工具延迟加载：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
