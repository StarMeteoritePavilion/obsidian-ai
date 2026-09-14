---
title: /compact之后Claude Code的上下文发生了什么变化？
source: https://www.bilibili.com/video/BV1JWEg6GEuv
author: 张司机在路上
created: 2026-06-08
tags:
  - AI
  - Claude Code
  - Anthropic
  - Compaction
  - 上下文压缩
  - Prompt Engineering
  - Context Engineering
  - 驾驭工程
  - 应用工程
---

# /compact之后Claude Code的上下文发生了什么变化？

Claude Code 的 `/compact` 不是简单删除旧消息，也不是只生成一段自由摘要。抓包案例显示，它先在原对话末尾注入一条严格的总结指令，让模型把长对话改写成结构化交接文档；随后丢弃生成过程中的 `<analysis>`，保留 `<summary>`，再与近期消息、相关文件和 Hook 上下文重新组装下一次请求。

## 从 87 条消息开始的压缩案例

案例要求 Claude Code 从空目录创建一个名为 `whateat` 的 Python CLI 工具，根据日期推荐当天吃什么。用户先给出整体推荐逻辑，并让湘菜、新疆菜和粤菜等偏好获得更高权重；后续继续添加菜单数据、推荐历史功能和单元测试。

完成这些工作后，对话已累积 87 条消息，上下文接近十万 Token。用户执行 `/compact` 时，Claude Code 在第 63 轮请求的原消息数组末尾加入一条以 `CRITICAL` 开头的 user 消息，要求模型停止继续工作，转而总结此前对话。

## 总结指令的三道约束

### 禁止调用任何工具

指令开头要求模型只返回文本，不得调用工具，并明确列出 Read、Bash、Grep、Glob、Edit 和 Write 等工具。中间再次说明工具调用会被拒绝并浪费当前轮次，结尾还重复提醒。相同限制共出现三次。

模型此时刚执行了数十轮任务，可能仍会尝试读取文件或验证结果。总结阶段的上下文已经逼近上限，额外工具往返可能使压缩无法完成；反复禁止工具，是为了强制模型从执行模式切换为总结模式。

### 固定 `<analysis>` 与 `<summary>` 的顺序

模型必须先在 `<analysis>` 中按时间顺序梳理对话，再在 `<summary>` 中生成正式交接内容，两段标签顺序固定且缺一不可。这不是通过 Thinking 配置开启的推理通道，而是应用层提示词直接规定的输出结构。

Claude Code 收到响应后，会在重新组装上下文前截去 `<analysis>`，只留下 `<summary>`。分析阶段帮助模型判断哪些信息应保留、哪些可以舍弃，以及后续继续工作需要什么状态；真正进入新上下文的是结构化摘要。

### 原样保留关键用户表述

总结模板要求第六节列出所有非工具结果的用户消息，并将安全相关指令或约束 `verbatim` 保留；第九节若存在下一步，还必须附上最近一次相关对话的 `direct quote`。这两处原文复制用于降低压缩后的意图偏移，特别是避免否定条件、安全边界和当前任务在摘要中被改写。

## 九节交接模板

`<summary>` 必须按照固定结构覆盖九类内容：

1. **Primary Request and Intent**：用户的明确请求与意图；
2. **Key Technical Concepts**：讨论过的技术概念、技术栈和框架；
3. **Files and Code Sections**：查看、修改或创建的文件与代码片段；
4. **Errors and Fixes**：遇到的错误、修复方法及用户反馈；
5. **Problem Solving**：已经解决的问题与仍在进行的排查；
6. **All user messages**：所有非工具结果的用户消息；
7. **Pending Tasks**：用户明确要求但尚未完成的任务；
8. **Current Work**：发起总结前正在进行的工作；
9. **Optional Next Step**：与最近工作直接相关的下一步。

总结请求还把 `max_tokens` 从 64,000 调整为 20,000，限制这份交接文档的最大长度。案例中，接近十万 Token 的历史最终被压缩为不到一万 Token，处在该上限以内。

## 压缩后的请求怎样重建上下文

执行 `/compact` 后的第一次请求是第 64 轮。System Prompt 没有变化，`messages` 数组则从 87 条缩减为 4 条；但这 4 条消息内部仍包含多个不同来源的 Content Block，不能把它们理解为只剩四段普通对话。

与继续工作直接相关的内容可分为三个核心组件：

- **Summary**：用九节模板压缩较早的对话历史；
- **MessagesToKeep**：保留压缩前最后一轮 Assistant 回复等近期现场；
- **Attachments**：重新载入近期工作所需的文件内容。

案例保留了添加单元测试后的 Assistant 回复，其中包括测试通过结果。Attachments 又恢复了多个相关测试文件的读取内容。除此之外，请求还包含项目 `CLAUDE.md`、MCP Server Instructions、SessionStart Hook、Local Command Caveat、Compacted Message 提示和压缩前最后一条用户消息等正常运行所需的上下文。

## 近期消息与文件不是简单截取

MessagesToKeep 不是机械保留最后 N 条消息。若近期存在一对 `tool_use` 与 `tool_result`，两者不能被切开；保留结果时必须同时带上对应调用，才能维持 API 消息结构合法。

Attachments 负责恢复工作材料。案例所示逻辑从最近修改过的历史记录中取出 5 个文件，单个文件最多保留 5,000 Token。源码画面还展示了 `fileAttachments`、`planAttachment` 和 `skillAttachment` 等不同附件路径，说明压缩后恢复的不只是摘要文本。

这也会产生重复：Summary 已经记录部分文件与代码，客户端仍可能重新附加相关文件内容。作者在阅读 Compact 相关源码后认为，这套组装方式更像为工作连续性叠加的多层补丁；如果 Summary 能完整恢复状态，额外附件本应不再必要，但实际摘要可能丢失细节，因此客户端继续保留近期回复和工作文件。

## `/compact` 的实质是生成工程交接文档

Summary 压缩远端历史，MessagesToKeep 保存就近现场，Attachments 恢复工作材料。三者共同回答一个问题：如果压缩后换成另一个模型实例，它能否沿着原任务继续工作。

因此，`/compact` 的核心不是“把文字变短”，而是把长对话改写成工程交接文档，再把近期状态和关键材料补回上下文。它能够显著减少历史 Token，却无法保证所有细节都被完整保留；禁止工具、固定结构、原文引用和附件恢复，分别用于约束压缩过程中的执行惯性、遗漏、意图漂移和现场丢失。
