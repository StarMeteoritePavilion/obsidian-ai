---
title: AI Agent 实践：Pydantic AI 工具调用与消息历史
source: https://www.bilibili.com/video/BV1UMVKzEESL
author: 隔壁的程序员老王
published: 2025-05-08
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - AI Agent
  - Pydantic AI
  - Function Calling
  - Gemini
  - 上下文工程
  - 应用工程
  - 资料摘要
---

# AI Agent 实践：Pydantic AI 工具调用与消息历史

原始资料：[[raw/sources/应用工程/AI Agent/原来写一个 AI Agent 这么简单|原来写一个 AI Agent 这么简单]]

## 核心结论

最小 AI Agent 是用户、模型与本地函数之间的应用层调度链。工具注册让模型知道可以请求哪些操作，但实际文件访问仍由 Agent 调用本地函数完成。Pydantic AI 的 `run_sync()` 调用默认彼此独立；跨调用上下文需要应用显式保存 `resp.all_messages()`，并在下一次调用时通过 `message_history` 回传。

## 示例链路

1. 应用实现 `read_file`、`list_files` 与 `rename_file`，只管理 `test` 目录中的示例文件。
2. 清晰的函数名、参数名、类型标注和 docstring 共同构成模型可见的工具说明。
3. 应用创建 `GeminiModel("gemini-2.5-flash-preview-04-17")`，通过 `.env` 和 `load_dotenv()` 加载 `GEMINI_API_KEY`。
4. `Agent` 接收模型、System Prompt 与工具列表；用户输入交给 `run_sync()`，最终结果从 `resp.output` 取得。
5. 首轮任务先列出并读取文件，判断其中的 Python、C 与 Go 代码；没有消息历史时，后续问题会触发重复读取。
6. 保存并回传完整消息历史后，模型可利用先前工具结果，直接调用 `rename_file` 添加扩展名。

## 工具契约

资料将模型类比为 API 使用者，因此工具契约必须可读。视频画面与配套代码显示，`read_file` 在失败时捕获异常并返回 `An error occurred: ...` 字符串；其 docstring 表述为文件不存在时返回错误提示。用户粘贴字幕中的 `FileNotFoundError` 不符合当期实现，不能据此把工具行为写成返回异常对象。

## 上下文与记忆边界

- `message_history` 是应用传给下一次模型调用的上下文，不是模型内部的跨请求持久记忆。
- 示例把历史保存在进程内列表中，程序退出后不会保留。
- 历史复用可以避免重复读取，但历史持续增长仍会带来窗口、注意力和成本问题；资料没有实现修剪、摘要或检索。
- 文件改名来自模型基于既有上下文提出的工具调用，不代表 Agent 自动维护了结构化文件状态。

## 工程边界与版本

- 示例只展示工具声明、同步调用与消息历史，没有实现权限隔离、执行沙箱、参数验证、失败恢复、持久状态、独立评判或自动停止。
- `.env` 不应提交到 Git，但这一做法本身不替代密钥轮换、最小权限和部署环境的秘密管理。
- 代码对应 2025 年 5 月的 Pydantic AI：使用 `GeminiModel` 与 `system_prompt`。当前文档已使用 `GoogleModel`，并更推荐 `instructions`；实际开发应按安装版本核对官方接口，不能用当前名称倒改历史资料。

## 关联

- 概念链路：[[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP]]
- 对话历史治理：[[wiki/sources/上下文工程：从 Prompt 到 Agent 上下文治理]]
- 上下文综合：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- 循环边界：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
