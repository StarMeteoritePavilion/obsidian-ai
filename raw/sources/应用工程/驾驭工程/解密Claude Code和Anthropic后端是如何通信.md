---
title: 解密Claude Code和Anthropic后端是如何通信
source: https://www.bilibili.com/video/BV1G2o5BqELx
author: 张司机在路上
created: 2026-04-24
tags:
  - AI
  - Claude Code
  - Anthropic
  - SSE
  - Prompt Caching
  - 上下文工程
  - 驾驭工程
  - 应用工程
---

# 解密Claude Code和Anthropic后端是如何通信

在 Claude Code 中输入 `hello`，模型只回复“Hello! How can I help you today?”，但这次调用实际处理了接近三万个输入 Token。用户可见的五个字母只占很小一部分，系统提示词、自动注入的项目上下文、工具定义与缓存内容才是主要开销。

## 用 claude-trace 记录通信

作者使用 Mario Zechner 开发的 `claude-trace`，拦截 Claude Code 与 Anthropic 服务器之间的 API 请求和返回值。该工具通过 Monkey Patch 劫持 Claude Code 命令行中的 `fetch` 函数，并在会话退出后生成可由浏览器查看的 HTML 记录。

资料还介绍了同一作者的 `cchistory`：它用于比较两个 Claude Code 版本之间的 System Prompt 和工具定义变化；`claude-trace` 则显示一次实际会话发送和接收了什么。资料参考的文章标题为 *cchistory: Tracking Claude Code System Prompt and Tool Changes*。

启动完整请求记录时，可以把平常使用的 `claude` 命令替换为：

```bash
claude-trace --include-all-requests
```

会话结束后，作者把 HTML 中的数据另存为 JSON，并按字段检查请求和 SSE 响应。

## `messages`：用户输入之外的五块注入内容

抓包请求使用 `claude-opus-4-7`。`messages` 数组看起来只有一条 `user` 消息，但它的 `content` 被包装成六个 Content Block。前五块均带有 `<system-reminder>` 标签，由 Claude Code 注入；最后一块才是用户的 `hello`。

第一块是 SessionStart Hook 注入的附加上下文。作者安装了 Superpowers，因此抓包中包含相应的 Skill 使用说明；没有安装这套配置的请求不会出现同一内容。这部分不属于 Claude Code 的固定内置提示。

第二块是延迟加载工具清单。它只列出尚未加载的工具名称，不携带每个工具的完整定义。模型真正需要某个工具时，再调用 `ToolSearch` 取得完整定义，以避免所有工具说明同时占用上下文。

第三块是 MCP Server 的使用说明，告诉模型当前提供了哪些 MCP 能力以及如何调用。作者的实例包含 Context7 文档 MCP。

第四块是当前可用 Skill 的完整列表，包括每项 Skill 的名称、触发条件和使用说明。

第五块包含项目中的 `CLAUDE.md`：项目结构、编写规则、工具偏好等指令会随请求交给模型，因此模型能够遵循当前项目规范。

最后的 `hello` Block 带有如下缓存控制：

```json
{
  "cache_control": {
    "type": "ephemeral",
    "ttl": "1h"
  }
}
```

前五个注入 Block 没有这个标记。资料据此把用户实际输入解释为缓存切割点：位于它之前的稳定内容可以写入并复用缓存，而当前输入本身每轮重新发送。

## `system`：四个系统文本块

`system` 不是一整段文字，而是由四个 Text Block 组成的数组。

第一块是计费头，记录 Claude Code 版本号与入口类型。第二块定义身份，抓包内容为：

> You are Claude Code, Anthropic's official CLI for Claude.

第三块是核心行为规则，覆盖运行环境、任务执行、谨慎操作、工具使用及表达风格。资料展示的规则包括：某些安全操作需要明确授权，删除文件与 Force Push 等动作需要谨慎；搜索内容使用 Grep，读取文件使用 Read，不以 Bash 中的同类命令替代；回答使用短句且不使用 Emoji。

第四块继续规定输出格式、推理强度和自动记忆，并携带 Shell 类型、工作目录等当前环境信息。自动记忆部分区分用户身份与偏好、用户反馈、哪些内容不应记录以及怎样写入记忆。

这些系统文本会随用户的每次对话进入请求。单看终端中的当前问题，无法看出模型实际收到的全部规则与环境。

## `tools`：定义能力也消耗上下文

本次抓包中，Claude Code 在 `tools` 字段定义了 10 个工具。资料展示的工具包括用于委派任务的 `Agent`、执行命令的 `Bash`、修改文件的 `Edit`、按文件名匹配的 `Glob`、搜索内容的 `Grep`、读取文件的 `Read`，以及 `ScheduleWakeup` 和 `ToolSearch`。

每个工具不只有名称，还带有参数 Schema 和完整使用说明。资料称，仅 `Bash` 的说明就超过一万个字符，其中包含何时使用命令、何时创建 Git Commit 以及怎样提交 Pull Request 等规则。因此，工具数量和说明长度也是 Agent 请求成本的一部分。

## Thinking 与 Effort 的内部口径冲突

请求中的 `thinking.type` 为 `adaptive`，表示由模型决定是否启动深度推理。画面里的 `output_config.effort` 明确显示为 `high`。

旁白则称 Effort 从 `low` 到 `xhigh` 共五档，并把 `xhigh` 说成当前默认和 `ultrathink` 的触发档位。这与同一抓包画面的 `high` 不一致，因此只能确认本次 JSON 使用 `adaptive` 与 `high`，不能由旁白认定该版本每次调用都固定使用 `xhigh`。

## 返回值是一串 SSE 事件

Claude Code 收到的不是一次性完整 JSON，而是 `text/event-stream` 格式的 Server-Sent Events（SSE）。本次响应按以下顺序推进：

1. `message_start` 返回消息 ID、模型名称和初始 Token 用量。
2. `content_block_start` 开启一个文本块。
3. `ping` 作为保持连接的心跳事件。
4. 两个 `content_block_delta` 分别推送 `Hello!` 和 ` How can I help you today?`。
5. `content_block_stop` 结束文本块。
6. `message_delta` 给出 `stop_reason: end_turn`，并更新最终用量。
7. `message_stop` 结束整个消息流。

终端中的逐步打印效果，正是多个 `content_block_delta` 依次到达的结果。`end_turn` 表示模型主动结束本轮回复。

## 一句 `hello` 的 Token 构成

`message_start` 中的初始 `usage` 为：

- `input_tokens`：6；
- `cache_creation_input_tokens`：14,145；
- `cache_read_input_tokens`：16,835；
- `output_tokens`：4。

`message_delta` 将最终 `output_tokens` 更新为 15。三个输入项合计为：

$$
6+14{,}145+16{,}835=30{,}986
$$

其中，6 个 Token 对应 `hello` 及其格式标记；14,145 个 Token 是本轮新写入缓存的内容；16,835 个 Token 从既有缓存读取。加上最终 15 个输出 Token，本轮总计量为 31,001 Token。

资料称缓存读取价格比普通输入低九成，因此连续对话不会把全部历史都按普通输入价格重新计费。这是视频发布时对 Anthropic 价格与当前抓包的解释；具体费用仍取决于实际模型、缓存策略和当时价格，不能把本次比例与 Token 构成外推到其他版本或会话。

这次抓包说明，Coding Agent 的一次模型调用远不只是用户输入与模型回复。Hook、延迟工具清单、MCP 说明、Skill、项目规则、系统行为约束、工具 Schema、环境、缓存读写和流式事件，共同构成 Claude Code 与 Anthropic 后端之间的通信。
