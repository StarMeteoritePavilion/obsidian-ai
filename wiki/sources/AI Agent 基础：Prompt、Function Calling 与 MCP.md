---
title: AI Agent 基础：Prompt、Function Calling 与 MCP
source: https://www.bilibili.com/video/BV1aeLqzUE6L
author: 隔壁的程序员老王
published: 2025-05-01
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - AI Agent
  - Prompt
  - Function Calling
  - MCP
  - 应用工程
  - 资料摘要
---

# AI Agent 基础：Prompt、Function Calling 与 MCP

原始资料：[[raw/sources/应用工程/AI Agent/10分钟讲清楚 Prompt, Agent, MCP 是什么|10分钟讲清楚 Prompt, Agent, MCP 是什么]]

## 核心结论

Prompt、Agent、Function Calling 与 MCP 属于不同层级：User Prompt 承载用户输入，System Prompt 承载角色与系统规则；模型负责产生回复或工具调用请求；Agent 负责组织模型调用、执行工具并回传结果；Function Calling 规范模型与 Agent 之间的工具调用结构；MCP 规范 Agent 与外部工具、资源和提示词服务之间的连接。

## 端到端关系

1. 用户请求进入 User Prompt，System Prompt 同时提供稳定背景和行为约束。
2. Agent 从本地注册或 MCP Server 取得工具说明。
3. Agent 通过自然语言格式约定或 Function Calling 向模型声明工具。
4. 模型返回工具调用请求，Agent 负责实际执行。
5. 工具结果经 Agent 返回模型，模型生成最终回复。

## 边界

- Agent 不等于模型。模型选择下一步，Agent 负责消息传递、工具执行和流程维持。
- Function Calling 不执行函数，只让模型以结构化方式提出调用请求。
- MCP 不直接规定模型推理方式，也不绑定具体模型；它连接 MCP Client 与 MCP Server，并暴露 Tool、Resource 和 Prompt。
- 资料关于厂商 API 不统一和开源模型支持情况的判断对应 2025 年 5 月，接入具体模型前需要重新核对当前官方文档。

配套实践资料进一步把这条概念链落到 Pydantic AI：本地函数经 `tools` 注册，模型提出调用请求，Agent 负责执行；多次 `run_sync()` 不会自动共享上一轮内容，应用必须保存并通过 `message_history` 回传消息记录。这说明“有工具”和“有跨调用上下文”是两个独立配置。（[[wiki/sources/AI Agent 实践：Pydantic AI 工具调用与消息历史|Pydantic AI 实践]]）

## 关联

- 提示词工程：[[wiki/syntheses/提示词工程：从单轮指令到生产规范]]
- 上下文工程：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 从 Prompt 到上下文治理：[[wiki/sources/上下文工程：从 Prompt 到 Agent 上下文治理]]
- 循环工程：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- AI Agent 框架：[[wiki/sources/AI Agent 框架选型：十大框架与五大范式]]
- Pydantic AI 实践：[[wiki/sources/AI Agent 实践：Pydantic AI 工具调用与消息历史]]
