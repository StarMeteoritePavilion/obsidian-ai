---
title: 上下文工程：从 Prompt 到 Agent 上下文治理
source: https://www.bilibili.com/video/BV1iweMzXEm2
author: 隔壁的程序员老王
published: 2025-08-21
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 提示词工程
  - 上下文工程
  - AI Agent
  - 应用工程
  - 资料摘要
---

# 上下文工程：从 Prompt 到 Agent 上下文治理

原始资料：[[raw/sources/应用工程/上下文工程/AI 提示词工程 上下文工程 15分钟弄懂！|AI 提示词工程 上下文工程 15分钟弄懂！]]

## 核心结论

Prompt Engineering 管理当前要求怎样表达，Context Engineering 管理当前模型调用能看到哪些信息。Agent 的 Tool Call 与 Tool Response 会持续扩张上下文，而用户要求通常只出现在开头；上下文工程通过笔记、修剪、摘要、RAG 外移和工具输出精简，降低中间信息淹没初始目标的风险。

## 从提示词到上下文

- System Prompt 承载角色、背景和系统规则，User Prompt 承载用户直接输入。
- Zero-shot 只给任务要求，Few-shot 提供输入输出示范，CoT 引导模型分步处理问题。
- 在资料采用的无持久状态请求模型中，Agent 或聊天服务器保存历史，并在新请求中把历史重新交给模型。
- 工具型 Agent 还会把工具说明、Tool Call 与 Tool Response 加入上下文；自主步骤越多，中间信息越容易压过原始目标。

Pydantic AI 的最小示例给出了这项职责的代码级证据：默认连续调用 `run_sync()` 时，后一次调用看不到前一次读取过的文件；应用保存 `resp.all_messages()` 并通过 `message_history` 回传后，模型才能复用此前的工具结果。这里的“记忆”属于上下文重建，不是模型自动形成的持久状态。（[[wiki/sources/AI Agent 实践：Pydantic AI 工具调用与消息历史|Pydantic AI 实践]]）

## 五类治理方法

1. 用笔记保存任务清单、完成状态和关键事实，并把笔记放在上下文的显眼位置。
2. 删除过旧消息，同时保留最初的 System Prompt 与 User Prompt。
3. 用精炼摘要替换旧历史，缩短长度但接受摘要可能遗漏细节的风险。
4. 把超长 Tool Response 存入临时向量库，用查询工具按需取回片段。
5. 在工具返回进入上下文前删除 HTML 等无关内容，从源头减少冗余。

## 边界

- Prompt 是上下文的一部分，也可以规定笔记和状态更新策略，但不能独自治理长程 Agent 的全部信息。
- 删除、摘要和外移都会引入信息损失；资料没有提供适用于所有任务的统一方案。
- 关于网页产品内置推理和上下文行为的描述对应 2025 年 8 月，具体接口应按当前产品与模型重新核对。

## 关联

- Prompt、Context 与 Harness 边界：[[wiki/sources/驾驭工程：Prompt、Context 与 Harness 的边界]]
- Agent 与工具接口：[[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP]]
- 工具调用与消息历史实践：[[wiki/sources/AI Agent 实践：Pydantic AI 工具调用与消息历史]]
- 提示词工程：[[wiki/syntheses/提示词工程：从单轮指令到生产规范]]
- 上下文工程：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- RAG 外移基础：[[wiki/sources/上下文工程：RAG 个人知识库基础架构]]
- 长上下文退化：[[wiki/sources/上下文工程：第七期 上下文是怎么坏掉的]]
