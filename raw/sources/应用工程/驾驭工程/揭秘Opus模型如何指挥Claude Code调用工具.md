---
title: 揭秘Opus模型如何指挥Claude Code调用工具
source: https://www.bilibili.com/video/BV1sJ9tBQEmr
author: 张司机在路上
created: 2026-04-30
tags:
  - AI
  - Claude Code
  - Anthropic
  - Opus
  - Tool Use
  - Bash
  - SSE
  - 驾驭工程
  - 应用工程
---

# 揭秘Opus模型如何指挥Claude Code调用工具

在 Tool Use 交互中，大模型负责决定调用哪个工具、生成哪些参数并解释执行结果；真正运行命令、修改文件或访问外部资源的是 Claude Code 客户端或 Anthropic 服务器。模型输出结构化指令，执行方完成动作，再把结果作为新消息交回模型。

## Tool Use 的职责分工

资料把 Tool Use 概括为“模型负责说，客户端负责做”。模型侧承担三项工作：决定调用哪个工具、确定参数以及读取结果后继续生成。Claude Code 客户端则真正执行命令、发起网络请求或读写文件。

两侧通过 `tool_use` 和 `tool_result` 往返：模型先发出 `tool_use`，Claude Code 完成动作后返回 `tool_result`。模型并不直接观察本地执行过程，只能读取客户端返回的结果文本。

## 用 Bash 检查 Git 改动

作者创建了一个 Git 项目，修改 `README.md` 和 `hello.py`，并新增未跟踪文件 `notes.md`。随后使用 `claude-tap` 拦截 Claude Code 的 API 请求，并向 Opus 4.7 输入：

> git有什么改动

Claude Code 调用 Bash 工具，在本地执行：

```bash
git status && echo "\n---DIFF---" && git diff
```

该工具调用的 Description 为 `Show git status and diff`。命令返回当前 `main` 分支的状态、两个已修改文件的 Diff，以及未跟踪文件；抓包工具自身生成的 `.traces/` 也出现在随后返回的状态中。

## 第一次请求：模型生成 `tool_use`

第一次 API 请求把用户任务、系统提示词和工具列表发送给 Opus。服务器通过 SSE 返回响应，其中工具调用对应的 `content_block_start` 具有以下字段：

- `type` 为 `tool_use`，表示当前 Content Block 是工具调用；
- `id` 是以 `toolu_01` 开头的本次调用唯一编号；
- `name` 为 `Bash`，指定客户端需要使用的工具；
- `input` 由后续 `input_json_delta` 逐段组成，最终包含完整的 `command` 与 `description`。

普通文本回复的 Content Block 使用 `type: text`，因此 `tool_use` 是不同的响应类型。该响应末尾的 `stop_reason` 为 `tool_use`，表示模型已经给出工具指令，正在等待客户端返回执行结果。资料以握手作类比，但这里的 `stop_reason` 是工具调用流程状态，并不是 HTTP 协议本身。

## 客户端执行并构造 `tool_result`

Claude Code 收到 `tool_use` 后完成两件事。首先，它从 `input.command` 取出命令，在用户电脑上执行 `git status` 和 `git diff`。其次，它把标准输出包装为 `tool_result`，放入第二次请求的 `messages`。

这个结果 Block 包含三个关键字段：

- `type` 为 `tool_result`；
- `tool_use_id` 与上一轮 `tool_use.id` 完全一致，用于把结果配对到对应调用；
- `content` 保存命令的原始文本输出。

第二次请求中的结果消息角色为 `user`。对模型而言，本地终端如何启动、命令怎样运行都不可见；它接收到的是由 Claude Code 返回的文本结果。

## 第二次响应：模型解释结果

Opus 读取 `tool_result` 后返回普通文本，说明当前位于 `main` 分支，`README.md` 和 `hello.py` 已修改，并列出 `.traces/` 与 `notes.md` 等未跟踪内容。此时 Content Block 的 `type` 为 `text`，`stop_reason` 为 `end_turn`，表示本轮工具调用闭环结束。

完整链路因此包含两次模型请求：

1. 第一轮把任务和工具定义交给模型，模型返回带参数的 `tool_use`。
2. Claude Code 在本地执行命令，把输出包装成 `tool_result`。
3. 第二轮把工具结果交回模型，模型生成自然语言总结并以 `end_turn` 结束。

## 客户端工具与服务器端工具

资料按执行位置区分两类工具。Bash、Edit 等 Client-executed Tool 在用户电脑上执行：模型发指令，Claude Code 客户端完成实际动作。WebSearch、WebFetch 等 Server-executed Tool 则由 Anthropic 服务器执行，客户端只接收结果。

本次案例展示的是客户端工具闭环。下一篇将继续拆解 WebSearch 这类服务器端工具如何完成搜索并把结果交回主模型。
