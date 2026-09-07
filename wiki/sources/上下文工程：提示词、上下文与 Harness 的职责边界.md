---
title: 上下文工程：提示词、上下文与 Harness 的职责边界
source:
  - https://www.bilibili.com/video/BV1pmMb6GEAM
  - https://www.bilibili.com/video/BV1iweMzXEm2
  - https://www.bilibili.com/video/BV1B6DiBbEf8
author:
  - 晴天AI实战
  - 隔壁的程序员老王
published: 2025-08-21
ingested: 2026-07-15
updated: 2026-09-07
tags:
  - AI
  - 上下文工程
  - 应用工程
  - 提示词工程
  - AI Agent
  - Harness
  - 资料摘要
---

# 上下文工程：提示词、上下文与 Harness 的职责边界

原始资料：[[raw/sources/应用工程/上下文与知识工程/提示词、上下文与 Harness：职责边界与基础实践|提示词、上下文与 Harness：职责边界与基础实践]]

## 核心结论

Prompt 主要表达任务，Context 管理当前调用可见的信息，Harness 用模型外的工作流、权限和检查约束执行。三者可以共享实现载体，分类应看主要职责；这是一套整理框架，不是统一行业标准。

## 主要内容

- 一次调用的上下文可包含系统指令、对话历史、外部知识、工具定义、用户输入和示例。
- 基础模型请求通常需要应用显式回传历史；工具调用还会不断加入调用请求和返回结果。
- 长任务通过任务笔记、旧消息修剪、历史摘要、长返回外移和工具输出精简控制窗口。
- Harness 在模型外预制工作流、限制目录权限，并用测试或实际状态检查结果。
- 删除、摘要和外移都会造成信息损失；模型升级也可能改变既有工程措施的收益，必须持续回归验证。

## 来源差异

三份资料对 Prompt 与 Context 的职责大体一致，对 Harness 和 Memory 的范围采用不同口径。合并稿保留差异，并删除了缺少模型、任务和评测配置的“新模型击败工程系统”个案作为技术证据。

## 关联

- [[wiki/syntheses/提示词工程：从单轮指令到生产规范]]
- [[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
