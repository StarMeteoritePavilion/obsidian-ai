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

当前原始资料：[[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 任务路径修订版）|Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code]]

历史版本：[[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code|初版（保留）]]。另保留 [[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 实践补充版）|实践补充版]]。另保留 [[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 权限澄清版）|权限澄清版]]。当前版本补齐目标路径输入步骤，视频发布日期不变。

## 核心结论

模型负责生成 Thought、工具请求或最终答案，Agent 主程序负责解析响应、执行函数、保存 Observation 并继续循环。工具让模型能够间接感知和改变外部环境，但系统提示词本身不提供执行权限。

ReAct 以 Thought、Action、Observation 和 Final Answer 组织逐步决策。本文的最小实现用 `while` 循环、消息历史和三个本地函数串联模型与环境，并在运行终端命令前增加人工确认。

Plan-And-Execute 把规划与执行分开：Plan 模型生成初始计划，执行 Agent 完成当前步骤，Re-Plan 模型依据结果返回新计划或最终答案。执行 Agent 内部仍可采用 ReAct，两种模式不是互斥层级。

## 演示边界

- DeepSeek 页面演示因没有单独的系统提示词入口，把系统提示词与用户任务合并提交；这不是规范 API 消息结构。
- 可运行示例使用同步返回，没有实现流式输出。
- 终端命令确认只覆盖一类显式风险；示例没有证明完整权限治理、沙箱、回滚、独立验收或生产可靠性。
- `openai/gpt-4o`、函数名、标签和项目结构对应作者 2025 年 7 月的当次代码画面，不表示当前产品接口。

## 目标目录与权限边界

`snake` 是任务面向的项目目录。固定源码中，目录参数用于存在性检查和提示词文件列表；`read_file`、`write_to_file` 直接使用传入路径，没有项目目录范围校验。`run_terminal_command` 未设置 `cwd`，程序也未切换工作目录；它继承启动进程的工作目录。命令执行前的人工确认只覆盖终端工具，不覆盖文件读写，也不构成沙箱。

因此，不能把“指定项目目录”写成“只允许在该目录操作”；实际访问仍受进程的操作系统权限等外部条件限制。依据与代码位置见 [[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 任务路径修订版）#目标项目目录不构成权限边界|源码边界核验]]，本结论来自静态核查，未执行越界读写实验。

## 实践入口与复现状态

- 作者源码为 [MarkTechStation/VideoCode](https://github.com/MarkTechStation/VideoCode)，固定提交 `27052e6db5b91d5f65e8de008f37a090471c77a1`；Python 3.12、uv、`OPENROUTER_API_KEY` 与已存在的 `snake` 目录是准备项。锁定的直接依赖为 Click 8.2.1、OpenAI SDK 1.91.0、python-dotenv 1.1.1。
- 完整获取、配置和启动步骤见 [[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 任务路径修订版）#按该版本准备并启动|当前修订版启动步骤]]。本轮只读核对代码并执行离线锁文件检查，没有安装项目依赖、调用模型或验收游戏。
- 新建空 `snake` 时，源码传给模型的文件列表为空，且没有单独传入项目目录。启动前用 `(cd snake && pwd -P)` 取得绝对路径，在“请输入任务：”中明确粘贴该路径并要求使用它；这只补足任务上下文，不增加权限隔离。具体输入示例见上述启动步骤。
- 作者所给 LangChain 原网址已跳转至 To-do list 中间件文档；原流程改用 [[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 任务路径修订版）#Plan-And-Execute 的官方实现与版本边界|固定提交的历史 Notebook]] 追溯。它使用 OpenAI 与 Tavily 两种凭据，依赖没有完整锁定，不能与作者的 ReAct 项目混用运行配置。

以上补充均核验于 2026-09-22，证据和版本选择依据见当前原始资料的“实践入口与运行前提”章节。

## 与长期 Loop 的衔接

跨资料比较：本文展示的是当前任务内的工具循环和动态重规划，不能仅凭这些流程认定已经具备长期 Loop 所需的外部触发、跨运行持久状态、独立验收及停止恢复机制。Re-Plan 负责调整计划或给出回答，其角色本身不等同于独立 Checker。此区分结合了 [[wiki/sources/循环工程：组件、搭建与上线检查|循环工程的运行条件]]，属于知识库综合判断，不是对原视频追加的原话。

比较表与接入顺序见 [[wiki/syntheses/AI Agent：从工具调用到可信行动#单任务执行循环与长期 Loop 的边界|单任务执行循环与长期 Loop 的边界]]。

## 关联

- [[wiki/syntheses/AI Agent：从工具调用到可信行动]]
- [[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
