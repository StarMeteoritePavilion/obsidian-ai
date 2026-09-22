---
title: AI Agent：ReAct 与 Plan-And-Execute 构建模式
source: https://www.bilibili.com/video/BV1TSg7zuEqR/
author: 马克的技术工作坊
published: 2025-07-22
ingested: 2026-09-22
updated: 2026-09-22
tags:
  - AI
  - AI Agent
  - ReAct
  - Plan-And-Execute
  - 应用工程
  - 资料摘要
---

# AI Agent：ReAct 与 Plan-And-Execute 构建模式

原始资料：[[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code|Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code]]

## 核心结论

模型负责生成 Thought、工具请求或最终答案，Agent 主程序负责解析响应、执行函数、保存 Observation 并继续循环。工具让模型能够间接感知和改变外部环境，但系统提示词本身不提供执行权限。

ReAct 以 Thought、Action、Observation 和 Final Answer 组织逐步决策。本文的最小实现用 `while` 循环、消息历史和三个本地函数串联模型与环境，并在运行终端命令前增加人工确认。

Plan-And-Execute 把规划与执行分开：Plan 模型生成初始计划，执行 Agent 完成当前步骤，Re-Plan 模型依据结果返回新计划或最终答案。执行 Agent 内部仍可采用 ReAct，两种模式不是互斥层级。

## 演示边界

- DeepSeek 页面演示因没有单独的系统提示词入口，把系统提示词与用户任务合并提交；这不是规范 API 消息结构。
- 可运行示例使用同步返回，没有实现流式输出。
- 终端命令确认只覆盖一类显式风险；示例没有证明完整权限治理、沙箱、回滚、独立验收或生产可靠性。
- `openai/gpt-4o`、函数名、标签和项目结构对应作者 2025 年 7 月的当次代码画面，不表示当前产品接口。

## 关联

- [[wiki/syntheses/AI Agent：从工具调用到可信行动]]
- [[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
