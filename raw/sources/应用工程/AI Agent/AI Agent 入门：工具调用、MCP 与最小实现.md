---
title: "AI Agent 入门：工具调用、MCP 与最小实现"
source:
  - "https://www.bilibili.com/video/BV1aeLqzUE6L"
  - "https://www.bilibili.com/video/BV1UMVKzEESL"
author:
  - "隔壁的程序员老王"
created: 2025-05-01
tags:
  - "AI"
  - "AI Agent"
  - "Prompt"
  - "Function Calling"
  - "MCP"
  - "应用工程"
  - "Pydantic AI"
  - "Gemini"
  - "上下文工程"
updated: 2026-09-07
---

# AI Agent 入门：工具调用、MCP 与最小实现

Agent 把用户请求、模型选择、工具执行和结果回传组织成循环。模型生成工具调用请求，真正读取文件、访问网络或修改数据的是应用中的函数或外部服务。本文先解释这条链路，再用 Pydantic AI 跑通一个只操作临时文件的最小示例。工具与调度通过离线检查，模型端未做在线验证。[^验证]

## Agent 各方职责

User Prompt 承载用户问题；System Prompt 承载应用预设的角色、规则和背景。这种消息分工有助于组织输入，但不能单靠一段提示词保证权限隔离。模型根据收到的上下文选择下一步；工具执行具体操作；Agent 负责发请求、校验参数、调用工具和回传结果。[^原一]

以查询文件为例：应用先告诉模型有哪些函数及其用途；模型请求列目录，Agent 执行后把文件名返回；模型再请求读取某个文件，收到内容后输出答案。失败也必须回传：路径不允许、文件不存在或参数错误，应得到明确错误或停止，而不是继续当作成功。

## Function Calling 与 MCP 的两段接口

Function Calling 是应用与模型 API 之间表达工具定义和调用请求的机制。原来源用 `name`、`description`、`parameters` 展示一种工具声明结构；这些键不是所有厂商 API 的统一外层格式。早期应用也可以在提示词里约定格式，再自行解析，但这仍不等于执行权限。[^原一]

MCP 连接宿主应用中的 Client 与提供能力的 Server。一个应用可以包含多个 Client，分别连接不同服务；它也可以直接调用本地函数，使用外部服务并不强制采用 MCP。按 **2026-07-28** 版规范，Server 的三类基本能力是：[^协议]

| 能力 | 作用 |
| --- | --- |
| Tools | 可调用操作，执行后返回结果。 |
| Resources | 可供应用或模型使用的上下文与数据。 |
| Prompts | 可复用的消息模板与工作流。 |

该版本标准传输包括 **stdio** 和 **Streamable HTTP**；不能把旧 HTTP+SSE 传输与新版 Streamable HTTP 混写成同一个协议。Streamable HTTP 在其协议流程内可使用 SSE，具体行为依该版本传输规范。[^传输]

一次网页搜索的完整链路是：用户提出问题 → 宿主中的 MCP Client 发现 Server 工具 → Agent 转换成模型 API 的工具定义 → 模型返回调用请求 → Agent 经 MCP 请求 Server → Server 执行搜索并返回内容 → Agent 把结果传回模型 → 模型生成答案。**下文只注册本地 Python 函数，没有实现 MCP Server。**

## 可安装环境与运行入口

本次实际环境为 Python **3.14.4**、`pydantic-ai-slim` **2.40.0**、`google-genai` **2.22.0**，运行于 macOS 26.4.1 arm64。框架使用官方 `FunctionModel` 离线替身；替身输出工具请求，真实 `Agent.run_sync` 负责参数校验、工具执行、消息回传与限制检查。`ALLOW_MODEL_REQUESTS=False` 防止测试误发在线请求。[^测试]

在 Python 3.14.4 环境中，把下面完整代码保存为 `agent_demo.py`，再运行：

```sh
python3 -m venv .venv
.venv/bin/python -m pip install 'pydantic-ai-slim[google]==2.40.0' 'google-genai==2.22.0'
.venv/bin/python agent_demo.py
```

需要精确复用本次传递依赖时，使用文末的完整依赖锁定文件。此示例直接从环境变量读取在线密钥，**不依赖 `python-dotenv`，不加载 `.env`**。

## 工具定义与完整离线示例

三个工具统一返回 `ToolResult`，成功和失败都具有相同字段类型。只接受临时目录直属文件名；拒绝目录、内部及外部符号链接、绝对路径和带父目录的路径。重命名通过不覆盖目标的硬链接创建后移除旧名称实现；目标存在时创建失败，原文件保持不变。测试目录由 `TemporaryDirectory` 唯一创建，不复用用户数据。

`retries=0` 控制校验失败后的重试；`UsageLimits(request_limit=4, tool_calls_limit=2)` 才负责一次运行最多四次模型请求、两次工具调用。工具返回结构化业务错误仍是一次已执行调用，也计入调用预算。达到限制时抛出 `UsageLimitExceeded` 并停止。[^限制]

```python
"""使用真实 Pydantic AI 调度器验证临时文件工具；默认完全离线。"""
import argparse
import importlib.metadata
import json
import os
import platform
import tempfile
from pathlib import Path

from pydantic import BaseModel
from pydantic_ai import Agent, models
from pydantic_ai.exceptions import UsageLimitExceeded, UnexpectedModelBehavior
from pydantic_ai.messages import ModelResponse, TextPart, ToolCallPart, ToolReturnPart
from pydantic_ai.models.function import FunctionModel
from pydantic_ai.usage import UsageLimits


class ToolResult(BaseModel):
    ok: bool
    message: str
    files: list[str] = []
    content: str = ''


def make_agent(base: Path, model):
    """三个工具仅处理本次临时目录直属的普通文件。"""
    base = base.resolve(strict=True)

    def checked(name: str) -> Path:
        if not name or Path(name).name != name or name in {'.', '..'}:
            raise ValueError('只接受临时目录内的单个文件名')
        path = base / name
        if path.is_symlink():
            raise ValueError('拒绝符号链接')
        if not path.resolve().is_relative_to(base):
            raise ValueError('拒绝目录越界')
        return path

    def read_file(name: str) -> ToolResult:
        """读取直属普通文件的 UTF-8 文本，失败时返回结构化错误。"""
        try:
            path = checked(name)
            if not path.is_file():
                raise ValueError('文件不存在或不是普通文件')
            return ToolResult(ok=True, message='读取成功', content=path.read_text(encoding='utf-8'))
        except (OSError, ValueError) as error:
            return ToolResult(ok=False, message=str(error))

    def list_files() -> ToolResult:
        """列出直属普通文件，不列目录和符号链接。"""
        try:
            names = sorted(p.name for p in base.iterdir() if not p.is_symlink() and p.is_file())
            return ToolResult(ok=True, message='列出成功', files=names)
        except OSError as error:
            return ToolResult(ok=False, message=str(error))

    def rename_file(name: str, new_name: str) -> ToolResult:
        """只重命名直属普通文件；目标存在、目录或符号链接均拒绝。"""
        try:
            source, target = checked(name), checked(new_name)
            if not source.is_file():
                raise ValueError('源文件不存在或不是普通文件')
            # 硬链接创建会在目标存在时失败，不使用可覆盖目标的 rename。
            os.link(source, target, follow_symlinks=False)
            try:
                source.unlink()
            except OSError:
                target.unlink()
                raise
            return ToolResult(ok=True, message='重命名成功')
        except (OSError, ValueError) as error:
            return ToolResult(ok=False, message=str(error))

    return Agent(model, tools=[read_file, list_files, rename_file], retries=0,
                 system_prompt='只管理所提供的临时文件工具。失败时说明原因，完成后停止。')


def limits():
    return UsageLimits(request_limit=4, tool_calls_limit=2)


def offline():
    models.ALLOW_MODEL_REQUESTS = False
    print(json.dumps({'Python': platform.python_version(), '系统': platform.platform(),
                      'pydantic-ai-slim': importlib.metadata.version('pydantic-ai-slim'),
                      '模型': 'FunctionModel：脚本化离线替身'}, ensure_ascii=False))
    with tempfile.TemporaryDirectory(prefix='m03-') as folder:
        base = Path(folder)
        (base / 'a').write_text('print(1)\n', encoding='utf-8')
        (base / '已有.txt').write_text('不得覆盖', encoding='utf-8')
        (base / '子目录').mkdir()
        with tempfile.TemporaryDirectory(prefix='m03-outside-') as outside:
            external = Path(outside) / '外部.txt'
            external.write_text('不得访问', encoding='utf-8')
            (base / '外链').symlink_to(external)
            (base / '内链').symlink_to(base / 'a')
            seen = []

            def invoke(tool, arguments, history=None):
                responses = []

                def scripted(messages, info):
                    if not responses:
                        if history:
                            assert messages[:len(history)] == history
                        seen.append(list(messages))
                        responses.append(True)
                        return ModelResponse(parts=[ToolCallPart(tool, arguments)])
                    returns = [p for m in messages for p in m.parts if isinstance(p, ToolReturnPart)]
                    value = returns[-1].content
                    if isinstance(value, BaseModel):
                        value = value.model_dump()
                    result = ToolResult.model_validate(value)
                    return ModelResponse(parts=[TextPart(result.model_dump_json())])

                agent = make_agent(base, FunctionModel(scripted))
                result = agent.run_sync('执行离线检查', message_history=history, usage_limits=limits())
                assert result.usage.tool_calls == 1
                return ToolResult.model_validate_json(result.output), result.all_messages()

            result, _ = invoke('list_files', {})
            assert result.ok and result.files == ['a', '已有.txt']
            print('通过：列目录；目录与符号链接不暴露')
            result, history = invoke('read_file', {'name': 'a'})
            assert result.ok and result.content == 'print(1)\n'
            first_count = len(history)
            result, second_history = invoke('rename_file', {'name': 'a', 'new_name': 'a.py'}, history)
            assert result.ok and not (base / 'a').exists() and (base / 'a.py').read_text() == 'print(1)\n'
            assert len(second_history) > first_count and second_history[:first_count] == history
            assert any(isinstance(p, ToolReturnPart) and p.tool_name == 'read_file'
                       for m in seen[-1] for p in m.parts)
            print('通过：读取、重命名、两轮完整消息历史回传')
            cases = [
                ('缺失读取', 'read_file', {'name': '缺失'}),
                ('越界路径', 'read_file', {'name': '../外部.txt'}),
                ('绝对路径', 'read_file', {'name': str(external)}),
                ('外部符号链接', 'read_file', {'name': '外链'}),
                ('内部符号链接', 'read_file', {'name': '内链'}),
                ('目标已存在', 'rename_file', {'name': 'a.py', 'new_name': '已有.txt'}),
                ('目录源', 'rename_file', {'name': '子目录', 'new_name': '新目录'}),
                ('目录目标', 'rename_file', {'name': 'a.py', 'new_name': '子目录'}),
                ('缺失重命名', 'rename_file', {'name': '缺失', 'new_name': '新文件'}),
                ('越界重命名', 'rename_file', {'name': 'a.py', 'new_name': '../外部.txt'}),
                ('符号链接目标', 'rename_file', {'name': 'a.py', 'new_name': '外链'}),
            ]
            for label, tool, arguments in cases:
                result, _ = invoke(tool, arguments)
                assert not result.ok and result.message
                print('通过：' + label + '拒绝；返回结构化错误')
            try:
                invoke('read_file', {'name': ['错误类型']})
            except UnexpectedModelBehavior:
                print('通过：参数类型错误被框架拒绝，retries=0 不重试')
            else:
                raise AssertionError('参数类型未被校验')
            assert (base / '已有.txt').read_text() == '不得覆盖'
            assert external.read_text() == '不得访问'
            assert (base / '子目录').is_dir() and not (base / '新目录').exists()
            assert (base / 'a.py').read_text() == 'print(1)\n'
            print('通过：原文件内容及目录完整保留')
            calls = []

            def endless(messages, info):
                calls.append(messages)
                return ModelResponse(parts=[ToolCallPart('list_files', {})])

            agent = make_agent(base, FunctionModel(endless))
            try:
                agent.run_sync('连续调用', usage_limits=limits())
            except UsageLimitExceeded as error:
                assert 'tool_calls_limit' in str(error)
                returns = [p for m in calls[-1] for p in m.parts if isinstance(p, ToolReturnPart)]
                assert len(returns) == 2 and len(calls) == 3
                print('通过：执行2次工具后，第3次在执行前被调用上限拦截')
            else:
                raise AssertionError('调用上限未生效')
            try:
                agent.run_sync('限制模型请求', usage_limits=UsageLimits(request_limit=1, tool_calls_limit=2))
            except UsageLimitExceeded as error:
                assert 'request_limit' in str(error)
                print('通过：第2次模型请求被请求上限拦截')
            else:
                raise AssertionError('模型请求上限未生效')
    print('全部离线检查通过；未调用在线模型 API')


def online(model_name):
    # 仅在用户主动选择在线入口时读取环境密钥；模型名由使用者核对服务后传入。
    from pydantic_ai.models.google import GoogleModel
    from pydantic_ai.providers.google import GoogleProvider
    key = os.environ.get('GEMINI_API_KEY')
    if not key:
        raise ValueError('在线入口需要 GEMINI_API_KEY 环境变量')
    with tempfile.TemporaryDirectory(prefix='m03-online-') as folder:
        base = Path(folder)
        (base / 'a').write_text('print(1)\n', encoding='utf-8')
        agent = make_agent(base, GoogleModel(model_name, provider=GoogleProvider(api_key=key)))
        history = []
        while True:
            text = input('请求（输入 exit 退出）：')
            if text == 'exit':
                break
            try:
                result = agent.run_sync(text, message_history=history, usage_limits=limits())
                history = result.all_messages()
                print(result.output)
            except UsageLimitExceeded as error:
                print('本轮已停止：', error)
                break


if __name__ == '__main__':
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument('--online-model', help='主动启用在线 Gemini；填入已核对可用的模型标识')
    args = parser.parse_args()
    if args.online_model:
        online(args.online_model)
    else:
        offline()
```

## 消息历史回传与 Gemini 接口

`run_sync` 不会自动把上一次调用的对话带进下一次。`result.all_messages()` 包含本次及传入的历史消息；下一轮用 `message_history=history` 显式回传。上面离线入口先读取 `a`，再带完整历史重命名为 `a.py`；断言会确认第二轮模型实际收到第一轮的工具结果。这验证的是历史传递机制，不是语言模型识别代码语言的能力。[^历史]

原来源在 **2025-05-08** 使用 `GeminiModel("gemini-2.5-flash-preview-04-17")` 和 `python-dotenv`。该写法仅记作历史接口，不能据此声明预览模型今天可用。新增入口依据已安装版本和官方文档使用 `GoogleModel`、`GoogleProvider(api_key=...)`；模型标识由使用者核对自己账号下实际可用的服务后通过 `--online-model` 传入，不预填一个未经在线验证的型号。[^Google]

离线默认入口不读取密钥。仅当使用者主动选择在线入口时才读取 `GEMINI_API_KEY`，密钥不能硬编码或提交到仓库。在线入口复用相同工具与限制，在独立临时目录中操作，输入 `exit` 退出。本次未调用此入口，未验证在线认证、模型可用性或真实模型的任务完成效果。

## 验证结果与能力边界

实际离线运行通过列目录、读取、重命名、文件不存在、参数类型错误、越界、绝对路径、内外符号链接、拒绝覆盖、拒绝移动目录、两轮历史回传，以及工具调用和模型请求上限检查。已有目标、源文件和外部文件内容保持不变；第三次工具调用在执行前被拦截。[^验证]

示例依赖支持硬链接的本地文件系统，不支持时会返回错误。它只用于单进程、私有临时目录，没有防御另一个拥有同等本机权限的进程在检查和操作之间替换文件，也没有持久化记忆、多用户授权、容器隔离、崩溃恢复或执行前审批。`history` 只在内存里保存，退出即丢失。这些边界不改变本例对目录、覆盖和工具调用预算的既有检查。

## 来源与版本

| 编号 | 准确标题 | 作者/机构 | 发布日期/版本 | URL或本地原件 | 定位 | 支持范围 |
| --- | --- | --- | --- | --- | --- | --- |
| 原一 | 10分钟讲清楚 Prompt, Agent, MCP 是什么 | 隔壁的程序员老王 | 2025-05-01（原稿属性） | [原视频](https://www.bilibili.com/video/BV1aeLqzUE6L) | 原稿“Agent”“一条完整的调用链” | 历史概念解释与搜索案例 |
| 原二 | 原来写一个 AI Agent 这么简单 | 隔壁的程序员老王 | 2025-05-08（原稿属性） | [原视频](https://www.bilibili.com/video/BV1UMVKzEESL) | 原稿“配置模型与密钥”“run_sync() 默认不会跨调用保留历史” | 历史 Gemini 示例与消息历史问题 |
| 协议 | Specification | MCP 项目 | 2026-07-28 | [协议](https://modelcontextprotocol.io/specification/2026-07-28) | Overview、Features | Host/Client/Server 与三类能力 |
| 传输 | Overview；Streamable HTTP | MCP 项目 | 2026-07-28 | [传输](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/index.md)；[HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http.md) | 标准传输及 HTTP 请求响应 | stdio、Streamable HTTP 与 SSE 边界 |
| 测试 | Testing | Pydantic | 2026-09-06读取；包2.40.0 | [官方文档](https://pydantic.dev/docs/ai/guides/testing/) | Unit testing with FunctionModel | 替身、模型请求禁用 |
| 限制 | UsageLimits | Pydantic | 2026-09-06读取；包2.40.0 | [API文档](https://pydantic.dev/docs/ai/api/pydantic-ai/usage/) | request_limit、tool_calls_limit | 模型请求与工具执行预算 |
| 历史 | Message History | Pydantic | 2026-09-06读取；包2.40.0 | [官方文档](https://pydantic.dev/docs/ai/core-concepts/message-history/) | all_messages、message_history示例 | 跨调用消息回传 |
| Google | Google | Pydantic | 2026-09-06读取；包2.40.0 | [官方文档](https://pydantic.dev/docs/ai/models/google/) | Installation、Configuration | GoogleModel 与 GoogleProvider 接口，不证明账号模型可用性 |
| 验证 | 本文正文中的完整离线程序 | 本文所列环境 | 2026-09-06 | 正文完整代码 | offline、make_agent、limits | 本文明确列出的工具及调度检查 |

[^原一]: 来源表“原一”；角色分工为概念说明，权限仍需应用实施。
[^协议]: 来源表“协议”的 Overview、Features；版本固定为2026-07-28。
[^传输]: 来源表“传输”；没有将旧版 HTTP+SSE 等同于新版传输。
[^测试]: 来源表“测试”；本次只替换模型返回逻辑，保留真实框架调度。
[^限制]: 来源表“限制”与本地运行日志；重试次数和调用预算分别配置。
[^历史]: 来源表“历史”及脚本中对第二轮输入消息的断言。
[^Google]: 来源表“原二”“Google”；历史代码与当前包接口分开表述。
[^验证]: 来源表“验证”；日志末行确认全部离线检查通过。
