---
title: Claude Code如何利用ToolSearch调用任意多个工具
source: https://www.bilibili.com/video/BV1Bd7X64EeF
author: 张司机在路上
created: 2026-06-03
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
---

# Claude Code如何利用ToolSearch调用任意多个工具

Claude Code 不必在每轮请求中加载所有工具的完整定义。ToolSearch 采用混合模式：少量常用工具以完整 Schema 进入 `tools` 数组，其余 Deferred Tools 只在 System Prompt 中列出名称；模型需要某项能力时，先查询 ToolSearch 取得定义，下一轮再正式调用目标工具。

这种机制同时处理两个问题：大量工具定义会消耗上下文并干扰选择，但动态改变 `tools` 数组通常又会破坏 Prompt Cache 前缀。`defer_loading` 与 `tool_reference` 让新增工具在对话末尾展开，从而保留既有缓存前缀。

## Deferred Tools 与 `tools` 数组的差异

抓包中的 System Prompt 列出 23 个 Deferred Tools，包括 `AskUserQuestion`、`WebSearch` 和两个 Context7 MCP 工具。这里仅有工具名称，没有 Description 或 JSON Schema。模型知道这些能力存在，却还不知道完整参数和调用方法。

同一请求的 `tools` 数组中有 10 个完整定义：`Agent`、`Bash`、`Edit`、`Glob`、`Grep`、`Read`、`ScheduleWakeup`、`Skill`、`ToolSearch` 和 `Write`。这些工具可以立即调用，但完整 Description 与 Schema 会直接消耗上下文。

ToolSearch 的作用是按需取得 Deferred Tool 的完整定义。它接收查询字符串与最大返回数量；需要精确选择工具时，可以使用 `select:<tool_name>` 形式。

## 代理环境为什么可能关闭 ToolSearch

作者使用 `claude-tap` 反向代理抓包。该工具会把 `ANTHROPIC_BASE_URL` 指向本地服务器，早期抓包因此出现所有工具定义一次性进入上下文、没有延迟加载的现象。

资料展示的 Claude Code 官方文档说明，Tool Search 默认开启，但在两类环境中默认关闭：一类是文档所列的 Vertex AI 兼容范围，另一类是 `ANTHROPIC_BASE_URL` 指向非 Anthropic 主机的代理。原因是多数代理不会转发 `tool_reference` Block。

可以用环境变量强制启用：

```bash
ENABLE_TOOL_SEARCH=true
```

官方配置表同时警告，`true` 会让 SDK 通过代理发送相关 Beta Header；若代理不支持 `tool_reference`，请求可能失败。因此，这项设置只适用于已经支持该协议的代理，不能把它当作任意第三方模型接口的通用兼容开关。

## AskUserQuestion 的两阶段调用

作者输入：“我现在好无聊呀，用交互式的方式给我三个出去玩的选项。”Claude Code 最终调用 `AskUserQuestion` 弹出选项。抓包显示，该过程不是一次普通工具调用，而是先加载定义、再使用工具。

### 第一步：客户端发送工具名与少量完整定义

Claude Code CLI 的请求包含 10 个完整工具定义、System Prompt 中 23 个 Deferred Tool 名称，以及用户问题。此时 `AskUserQuestion` 只有名称，不在 `tools` 数组中。

### 第二步：模型请求 ToolSearch

模型判断任务需要向用户展示选项，于是先返回一个针对 `ToolSearch` 的 `tool_use`：

```json
{
  "query": "select:AskUserQuestion",
  "max_results": 1
}
```

ToolSearch 根据名称在本地工具定义中匹配 `AskUserQuestion`，读取其完整 Description、Input Schema 和参数约束。

### 第三步：下一轮追加工具定义与引用

Claude Code CLI 发起下一次请求。`tools` 数组从 10 个增加到 11 个，新加入的 `AskUserQuestion` 带有：

```json
{
  "defer_loading": true
}
```

对话历史末尾还追加一个 `tool_result`，其内容为：

```json
{
  "type": "tool_reference",
  "tool_name": "AskUserQuestion"
}
```

这个 Reference 表示工具已经加载。服务端检测到后，会把相应工具定义以内联形式展开，使模型在后续上下文中看到完整说明。

### 第四步：模型正式调用目标工具

模型收到包含完整 Schema 的新请求后，才发起 `AskUserQuestion` 工具调用，并在 `questions` 参数中给出三个出行选项。

因此，ToolSearch 比普通工具调用多一次模型往返：第一次向客户端索取工具说明，第二次才使用该工具完成任务。

## 动态增加工具为什么没有破坏缓存

工具定义通常位于 Prompt Cache 前缀中。若 `tools` 数组从 10 项直接变为 11 项，变更点之后的缓存通常会失去匹配。

资料展示的 *defer_loading and cache preservation* 机制采用两步处理：

1. 带 `defer_loading: true` 的新增工具不计入原有缓存前缀的 Hash，因此不会改变既有 `tools` 前缀。
2. `tool_reference` 被追加到对话历史末尾；服务端将其展开为完整工具定义，模型仍然能够读取该工具。

新增信息只出现在原请求末尾，缓存前缀保持不变。系统因此同时保留 Prompt Cache 命中和完整工具能力。该机制依赖 Anthropic 对 `defer_loading` 与 `tool_reference` 的协议处理，不能用普通客户端字段拼接直接替代。

## Anthropic 给出的 Token 对照

Anthropic 于 2025-11-24 发布的 *Introducing advanced tool use on the Claude Developer Platform* 给出一组五服务器示例：

- GitHub：35 个工具，约 26K Token；
- Slack：11 个工具，约 21K Token；
- Sentry：5 个工具，约 3K Token；
- Grafana：5 个工具，约 3K Token；
- Splunk：2 个工具，约 2K Token。

五组共 58 个工具，在对话开始前便消耗约 55K Token；文章还称 Anthropic 内部见过优化前达到 134K Token 的工具定义。

同一文章把传统方式与 Tool Search 对比：

- 传统方式为 50 多个 MCP Tool 预先加载约 72K Token，加上对话历史和 System Prompt 后，在实际工作开始前总上下文约 77K Token。
- Tool Search 只预载自身定义，约 500 Token；按需发现 3～5 个相关工具再使用约 3K Token，总上下文约 8.7K Token，保留 95% 的上下文窗口。

文章将这种变化概括为 Token 用量降低 85%，同时保持全部工具可访问。对应 MCP 评估中，Opus 4 的工具选择准确率从 49% 提升到 74%，Opus 4.5 从 79.5% 提升到 88.1%。这些数字属于文章所述 MCP 评估与工具库配置，不代表其他模型、工具描述或任务分布上的固定收益。

## 工具更少，能力不必更少

ToolSearch 不是删除工具，而是把完整定义从“全部预载”改成“名称可见、Schema 按需展开”。初始上下文只承担发现成本，真正相关的少量工具才承担完整定义成本。

这项设计也缩小了模型的初始选择空间。资料中的评估显示，工具定义减少后选择准确率反而提高；其结论应限定在对应实验，不应泛化为工具越少必然越准确。长期可用的工程原则是：保留能力目录，延迟加载昂贵定义，并让协议层同时维护缓存前缀与工具可见性。
