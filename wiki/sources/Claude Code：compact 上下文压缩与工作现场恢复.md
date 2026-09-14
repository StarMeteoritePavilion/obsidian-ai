---
title: Claude Code：/compact 上下文压缩与工作现场恢复
source: https://www.bilibili.com/video/BV1JWEg6GEuv
author: 张司机在路上
published: 2026-06-08
ingested: 2026-09-14
updated: 2026-09-14
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
  - 资料摘要
---

# Claude Code：/compact 上下文压缩与工作现场恢复

原始资料：[[raw/sources/应用工程/驾驭工程/compact之后Claude Code的上下文发生了什么变化？|/compact之后Claude Code的上下文发生了什么变化？]]

## 核心结论

资料中的 `/compact` 先让模型把完整历史改写成九节结构化交接文档，再以 Summary、MessagesToKeep 和 Attachments 重新组装下一次请求。它不是只保留一段摘要：较早历史被压缩，近期消息保持协议完整，相关文件则重新进入上下文。

## 总结请求

案例中的 `whateat` Python CLI 任务累积 87 条消息、接近十万 Token。第 63 轮请求在原消息末尾注入一条以 `CRITICAL` 开头的 user 消息，并把 `max_tokens` 从 64,000 调整为 20,000。

提示词使用三道约束：三次禁止任何工具调用；要求先输出 `<analysis>`、再输出 `<summary>`；要求安全相关用户约束 `verbatim` 保留，并在下一步中附上最近用户请求的 `direct quote`。九节模板依次覆盖请求意图、技术概念、文件代码、错误修复、问题解决、全部用户消息、待办、当前工作和下一步。

模型返回两段内容后，Claude Code 丢弃 `<analysis>`，只把 `<summary>` 放回上下文。案例中的历史由接近十万 Token 压缩到不到一万 Token。

## 工作现场恢复

压缩后的第 64 轮请求保持 System Prompt 不变，`messages` 从 87 条缩减到 4 条，但消息内部包含多个 Content Block。Summary 负责较早历史，MessagesToKeep 保留近期现场，Attachments 恢复工作材料；项目 `CLAUDE.md`、MCP Server Instructions、SessionStart Hook 和最后一条用户消息等运行上下文也继续存在。

MessagesToKeep 会保护 `tool_use`／`tool_result` 配对，不能只保留结果。Attachments 从最近修改记录中取出 5 个文件，单个文件最多 5,000 Token；源码画面还显示 `fileAttachments`、`planAttachment` 和 `skillAttachment` 等路径。

## 边界与作者判断

Summary 与重新附加的文件可能包含重复内容。作者阅读 Compact 相关源码后的判断是，这些附件像是为弥补摘要无法独立恢复工作连续性而增加的补丁；这是作者对当次实现的评价，不应改写为所有 Claude Code 版本的固定设计结论。

资料没有提供对应 Claude Code 版本号、完整抓包文件、源码仓库或 Commit。87 条消息、接近十万 Token、压缩后 4 条消息、5 个文件和每文件 5,000 Token 都属于作者当次案例与代码画面，不能外推为其他版本的固定阈值。

## 关联

- 上一篇：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- 下一篇：[[wiki/sources/Claude Code：权限规则、Permission Mode 与本地放行]]
- 后续 Skill 加载：[[wiki/sources/Claude Code：Skill 渐进式披露与第三方执行边界]]
- 通用上下文压缩：[[wiki/sources/上下文工程：第五期 上下文工程压缩]]
- Claude Code Runtime：[[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
