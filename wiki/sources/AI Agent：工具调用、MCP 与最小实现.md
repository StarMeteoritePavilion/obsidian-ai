---
title: AI Agent：工具调用、MCP 与最小实现
source:
  - https://www.bilibili.com/video/BV1aeLqzUE6L
  - https://www.bilibili.com/video/BV1UMVKzEESL
author:
  - 隔壁的程序员老王
published: 2025-05-01
ingested: 2026-09-04
updated: 2026-09-07
tags:
  - AI
  - AI Agent
  - MCP
  - Pydantic AI
  - 应用工程
  - 资料摘要
---

# AI Agent：工具调用、MCP 与最小实现

原始资料：[[raw/sources/应用工程/AI Agent/AI Agent 入门：工具调用、MCP 与最小实现|AI Agent 入门：工具调用、MCP 与最小实现]]

## 核心结论

模型负责提出回复或工具调用，Agent 负责组织消息、执行工具并回传结果。Function Calling 规范模型与 Agent 之间的调用结构；MCP 规范客户端连接提供 Tool、Resource 与 Prompt 的服务。文中的本地文件示例注册普通 Python 函数，没有实现 MCP Server。

## 从概念到可运行示例

- User Prompt 承载用户请求，System Prompt 或 instructions 承载稳定规则；具体字段取决于安装版本。
- 工具声明必须包含准确名称、参数和返回契约，执行权限仍由应用控制。
- Pydantic AI 示例把列目录、读文件和改名工具限制在临时目录；拒绝目录、覆盖目标、越界路径和外部符号链接。
- 多次模型调用不会凭空共享状态。应用保存 `all_messages()` 并通过 `message_history` 回传，才形成跨调用上下文。
- 离线验收覆盖成功路径、缺失文件、越界、符号链接、调用上限和两轮历史；在线 Gemini 调用没有纳入本次实测。

## 边界

工具可用不等于系统具备权限治理、持久状态、独立评判、恢复或自动停止。示例解释一条最小调度链，生产系统仍需单独设计这些 Harness 能力。文中同时保留2025年历史接口和2026年核验环境，不能把历史模型字符串解释为当前可用性保证。

## 关联

- [[wiki/syntheses/AI Agent：从工具调用到可信行动]]
- [[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
