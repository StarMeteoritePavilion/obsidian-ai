---
title: 原来写一个 AI Agent 这么简单
source: https://www.bilibili.com/video/BV1UMVKzEESL
author: 隔壁的程序员老王
created: 2025-05-08
tags:
  - AI
  - AI Agent
  - Pydantic AI
  - Function Calling
  - Gemini
  - 上下文工程
  - 应用工程
---

> AI Agent 实践独立专题

AI Agent 可以理解为连接用户、AI 模型与功能函数的调度程序。模型本身不能直接读取本地硬盘；应用需要先实现文件操作函数，再把函数的信息注册给 Agent。Agent 将这些信息通过 System Prompt 或 Function Calling 告诉模型，模型返回工具调用请求，Agent 执行对应函数并继续传递结果。

下面用 Pydantic AI 和 Gemini 实现一个最小 Agent：它只管理 `test` 目录中的四个示例文件。这些文件分别包含 Python、C 与 Go 代码，最初没有扩展名。

## 先把本地能力写成工具

示例预先实现了三个普通的文件操作函数：

- `read_file(name: str) -> str`：读取文件内容。
- `list_files() -> list[str]`：列出目录中的文件。
- `rename_file(name: str, new_name: str) -> str`：修改文件名。

这些函数与 AI 没有直接关系。关键在于，要让模型仅凭工具说明就能判断何时以及怎样调用它们。函数名和参数名应清晰表达用途，参数与返回值应具有明确的类型标注；如果函数行为无法仅凭名称说明，还应补充 docstring。

例如，`read_file` 的 docstring 说明文件不存在时会返回错误提示。实现中捕获文件读取异常，并把错误写入返回字符串，而不是返回 `FileNotFoundError` 异常对象。对模型而言，这些名称、类型和文档共同构成工具 API 的使用说明。

```python
from pathlib import Path

base_dir = Path("./test")


def read_file(name: str) -> str:
    """Return file content. If not exist, return error message."""
    try:
        with open(base_dir / name, "r") as f:
            return f.read()
    except Exception as e:
        return f"An error occurred: {e}"
```

可以把模型当成使用 API 的程序员：应用提供可调用函数和足够明确的 API 文档，模型再根据任务选择调用方式。

## 配置模型与密钥

示例使用 Pydantic AI，并创建代表 Gemini 模型的 `GeminiModel` 实例：

```python
from pydantic_ai.models.gemini import GeminiModel

model = GeminiModel("gemini-2.5-flash-preview-04-17")
```

API Key 不应直接写入代码。Gemini 接口从 `GEMINI_API_KEY` 环境变量读取密钥；为了避免每次手动设置，可以把它保存在 `.env` 文件中，再使用 `python-dotenv` 加载：

```python
from dotenv import load_dotenv

load_dotenv()
```

`.env` 包含敏感信息，不应提交到 Git 仓库。

## 创建 Agent 并注册工具

模型准备好后，创建 `Agent` 对象。第一个参数指定模型，`system_prompt` 定义模型角色，`tools` 列出允许模型请求调用的本地函数：

```python
from pydantic_ai import Agent

agent = Agent(
    model,
    system_prompt="You are an experienced programmer",
    tools=[tools.read_file, tools.list_files, tools.rename_file],
)
```

至此，模型负责根据请求选择下一步，Agent 负责模型通信和工具调度，本地函数负责实际访问文件系统。

## 接收请求并执行工具

主程序通过标准输入接收用户请求，再将请求交给 `agent.run_sync()`。Agent 会根据模型返回的工具调用请求执行函数，必要时把结果继续交给模型，最后将 `resp.output` 打印给用户。

```python
def main() -> None:
    while True:
        user_input: str = input("Input: ")
        resp = agent.run_sync(user_input)
        print(resp.output)
```

第一次测试要求 Agent 判断 `a`、`b`、`c`、`d` 四个文件分别使用什么编程语言。模型先调用 `list_files` 获取文件列表，再逐个调用 `read_file` 读取内容，最后判断其中包含 Python、C 与 Go 代码。

## `run_sync()` 默认不会跨调用保留历史

随后追问 `b` 文件的功能时，Agent 又读取了一次 `b`。原因不是模型没有理解文件，而是每次调用 `agent.run_sync()` 默认相互独立：新的调用不会自动获得上一次的对话，也不知道此前执行过哪些工具。

如果希望后续调用使用先前上下文，应用需要显式保存完整消息记录，并在下一次调用时通过 `message_history` 传回：

```python
def main() -> None:
    history = []
    while True:
        user_input: str = input("Input: ")
        resp = agent.run_sync(user_input, message_history=history)
        history = list(resp.all_messages())
        print(resp.output)
```

加入消息历史后，先让 Agent 识别各文件的语言，再要求它根据内容修改文件名。模型可以直接利用此前读取过的内容调用 `rename_file`，为文件添加对应扩展名，不必再次调用 `read_file`。最终，`test` 目录中的文件名被正确修改。

这里的“记住”是应用层保存和回传消息历史的结果，并不是模型获得了独立的持久记忆。程序退出后，这个内存列表也会消失。

## 这个最小示例说明了什么

最基础的 Agent 链路由四部分组成：用户请求、模型决策、工具执行和消息回传。注册工具使模型能够提出操作请求，`message_history` 则让后续模型调用重新看到先前的对话与工具结果。

这段代码用于解释核心机制，并不是完整的生产 Agent。它没有实现持久化记忆、细粒度权限、参数校验、执行隔离、失败恢复、独立验证或自动停止等机制。工具注册只提供能力入口，不会自动补齐这些工程边界。
