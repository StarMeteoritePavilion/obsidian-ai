---
title: 10分钟讲清楚 Prompt, Agent, MCP 是什么
source: https://www.bilibili.com/video/BV1aeLqzUE6L
author: 隔壁的程序员老王
created: 2025-05-01
tags:
  - AI
  - AI Agent
  - Prompt
  - Function Calling
  - MCP
  - 应用工程
---

> AI Agent 基础独立专题

Prompt、Agent、Function Calling 与 MCP 处于不同层级。Prompt 负责向模型传递信息；Agent 负责连接用户、模型与工具并组织执行；Function Calling 规范模型怎样提出工具调用；MCP 规范 Agent 怎样连接外部工具、资源和提示词。它们不是彼此替代的概念，而是共同组成一条完整的 AI 应用链路。

## User Prompt 与 System Prompt

资料从 2023 年用户面对 GPT 聊天框的体验讲起。用户向模型发送消息，模型根据输入生成回复。用户直接提出的问题或表达的内容称为 User Prompt，即用户提示词。

同一句话由不同的人回答，结果会受到其经验、角色和语气影响。模型如果没有获得类似背景，只能给出相对通用的回答。最直接的做法，是把角色设定和用户真正想说的内容放在同一条 User Prompt 中，但角色说明会与用户消息混在一起。

System Prompt 将这两类信息分开。它主要承载模型角色、性格、背景、语气和其他系统预设信息。每次发送 User Prompt 时，系统会把 System Prompt 一并交给模型。网页聊天产品中的 System Prompt 通常由系统预设；资料以 ChatGPT 的 Customize ChatGPT 为例，说明用户偏好也可以成为系统提示的一部分。

## Agent：把模型回复连接到实际行动

仅有聊天模型时，模型可以回答问题或说明做法，却不会自动完成电脑上的实际操作。资料以 AutoGPT 为例说明 Agent 怎样补上这一层。

如果希望 AutoGPT 管理本地文件，需要先提供目录查询、文件读取等函数，并注册这些函数的功能描述和使用方法。Agent 随后完成一条循环：

1. 告诉模型当前有哪些工具、工具的用途，以及模型应以什么格式请求调用。
2. 把工具信息和用户请求一起发送给模型。
3. 解析模型返回的工具调用请求并执行对应函数。
4. 把函数结果交回模型，由模型决定下一步操作。
5. 重复上述过程，直到任务完成。

在这套表述中，AI Agent 是用户、模型和工具之间的协调程序，提供给模型调用的函数或服务称为 Agent Tool。Agent 不等于模型本身：模型负责根据上下文选择下一步，Agent 负责传递消息、执行工具和维持流程。

## 从 System Prompt 约定到 Function Calling

早期 Agent 可以把工具说明与返回格式直接写入 System Prompt，让模型按照自然语言约定输出函数调用请求。但模型是概率模型，仍可能返回不符合格式的内容。部分 Agent 会在解析失败时自动重试，资料以 Cline 为例说明这种方式仍在使用。

Function Calling 把工具描述和调用格式进一步标准化。资料中的示例使用 JSON 对象定义工具：工具名写入 `name`，功能说明写入 `description`，参数定义写入 `parameters`。工具定义从 System Prompt 中分离，模型请求工具时也使用约定的结构。

统一结构使模型厂商能够针对工具调用场景训练和检测模型输出，也降低 Agent 端解析与重试的负担。资料同时指出，2025 年不同厂商的 API 定义并不统一，一些开源模型也不支持 Function Calling，因此基于 System Prompt 的约定与 Function Calling 在当时仍然并存。

Function Calling 处理的是 Agent 与模型之间的通信：Agent 向模型声明工具，模型返回调用请求，Agent 执行工具，再把结果交回模型。模型提出调用，不等于模型亲自执行了函数。

## MCP：连接 Agent 与外部能力

如果 Agent 和工具位于同一程序中，直接调用函数即可。但浏览网页等通用能力可能被多个 Agent 复用。把工具复制到每个 Agent 中会增加重复实现，于是可以把工具作为独立服务统一托管。

MCP 是 Agent 与工具服务之间的通信协议。运行工具服务的一端称为 MCP Server，连接服务的 Agent 一端称为 MCP Client。协议定义双方怎样通信，以及 Client 怎样查询 Server 提供的能力与参数格式。

MCP Server 可以提供三类内容：

- **Tool**：供 Agent 调用的函数或操作。
- **Resource**：文件读取等数据资源。
- **Prompt**：供 Agent 使用的提示词模板。

MCP Server 与 Agent 可以运行在同一台机器上，通过标准输入输出通信；也可以部署在网络上，通过 HTTP 通信。MCP 为 AI 应用设计，但协议本身不绑定具体模型。它负责管理 Agent 可连接的工具、资源和提示词，不决定 Agent 使用哪个模型。

## 一条完整的调用链

一次需要网页检索的请求可以串联所有概念：

1. 用户问题进入 User Prompt。
2. 作为 MCP Client 的 Agent 从 MCP Server 获取工具信息。
3. Agent 把工具定义转换为 System Prompt 中的自然语言约定，或者模型 API 所需的 Function Calling 格式。
4. Agent 将工具信息与 User Prompt 一起发送给模型。
5. 模型生成调用网页搜索工具的请求。
6. Agent 通过 MCP 调用 Server 中的网页工具。
7. 工具访问目标网页，把内容返回给 Agent。
8. Agent 将工具结果交回模型，模型据此生成最终回复。
9. Agent 把回复展示给用户。

这条链路揭示了几个容易混淆的边界：System Prompt 与 User Prompt 都进入模型输入；Function Calling 约束模型如何请求工具；MCP 连接 Agent 与工具服务；Agent 则把用户输入、模型调用、工具执行和结果回传组织成完整流程。

## 从理解概念开始参与变化

AI 名词快速增加容易带来焦虑，但理解概念之间的职责边界，可以把模糊的技术冲击还原为可分析的系统结构。面对持续发生的技术变化，与其只被动接受工具改变工作和生活，不如从理解一个 Agent 开始，主动辨认自己正在使用的模型、接口、协议和执行程序。
