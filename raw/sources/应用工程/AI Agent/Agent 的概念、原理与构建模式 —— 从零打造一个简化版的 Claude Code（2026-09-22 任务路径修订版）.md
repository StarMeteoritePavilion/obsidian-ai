---
title: Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 任务路径修订版）
source: https://www.bilibili.com/video/BV1TSg7zuEqR/
author: 马克的技术工作坊
created: 2025-07-22
tags:
  - AI
  - AI Agent
  - ReAct
  - Plan-And-Execute
  - 应用工程
---

# Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 任务路径修订版）

> 从工具调用出发，理解 ReAct 与 Plan-And-Execute 两种 Agent 构建模式

大模型能够回答问题、生成代码和完成逻辑推理，但模型本身不能直接感知或改变外部环境。以编程任务为例，模型可以生成一个贪吃蛇游戏的代码，却不能仅凭这次回答把代码写入本地文件；已有项目代码如果没有进入上下文，模型也无法自行读取。

工具弥补了这一缺口。读取文件、写入文件、列出目录和运行终端命令等工具，相当于大模型的感官和四肢。把大模型与一组工具组织成能够感知并改变外部环境的智能程序，就形成了 Agent。

## 从模型到 Agent

Agent 的类型取决于它能够使用的工具和需要完成的任务。编程 Agent 可以读取项目、修改代码并运行程序；其他 Agent 可以制作演示文稿或执行深度搜索。Cursor 是编程 Agent 的例子：用户提交任务后，它调用模型和工具持续处理，直至完成任务。Manus 的示例则先生成计划，再搜索和浏览网页，最后把手机性能与拍照能力的比较整理成页面。

这两个例子共同说明，模型只负责生成判断与行动请求，真正改变外部环境的是 Agent 中负责执行工具和串联流程的程序。

## ReAct：思考、行动、观察

ReAct 是 Reasoning and Acting 的缩写，即“推理与行动”。该模式最初由 2022 年 10 月的一篇论文提出。视频发布时，作者将其视为应用最广泛的 Agent 运行模式之一。

一个完整的 ReAct 循环包含四类输出：

1. **Thought**：模型分析当前任务，判断下一步是否需要调用工具。
2. **Action**：模型提出工具调用请求，包括工具名称和参数。
3. **Observation**：Agent 执行工具，把结果返回给模型。
4. **Final Answer**：模型判断不再需要工具时，输出最终答案并结束流程。

如果 Observation 仍不足以解决任务，模型会继续生成新的 Thought 和 Action。循环会一直进行到模型返回 Final Answer。

### 系统提示词规定运行协议

在本文演示的实现中，ReAct 的主要运行协议写在系统提示词中。系统提示词与用户问题一起发送给模型，用于规定角色、运行规则和环境信息。演示中的提示词由五部分组成：

- 职责描述；
- 示例；
- 可用工具；
- 注意事项；
- 环境信息。

职责描述要求模型把任务分解成多个步骤，每一步先输出 `<thought>`，再通过 `<action>` 请求工具；工具结果通过 `<observation>` 返回；信息足够时输出 `<final_answer>`。示例进一步展示单次和多次工具调用，可用工具部分列出读文件、写文件和运行终端命令等能力，环境信息则包括操作系统、当前目录和目录中的文件。

系统提示词并不让模型获得直接执行能力。模型输出 `<action>` 只表示请求调用工具，真正解析请求、执行函数并返回 Observation 的仍是 Agent 主程序。

### 用对话模拟 ReAct

演示任务是使用 HTML、CSS 和 JavaScript 编写贪吃蛇游戏，并把代码分别保存到不同文件。由于当时使用的 DeepSeek 对话页面没有单独提交系统提示词的位置，演示把系统提示词与用户任务合并为一条消息。这是页面条件下的临时处理；规范的 API 调用应把系统提示词与用户消息分开传递。

模型先请求 `write_to_file` 写入 `index.html`。演示者手动返回“写入成功”作为 Observation，模型随后请求写入 CSS 和 JavaScript 文件。三个文件完成后，模型返回 Final Answer。这个过程显示了 ReAct 的基本节奏：Thought、Action、Observation 反复出现，直到任务完成。

## 构建一个可运行的 ReAct Agent

仅靠人工回复 Observation 仍然只是模拟。作者随后在同一套系统提示词外增加工具执行与循环代码，构建出一个可以实际读写文件的 ReAct Agent。项目包含 `agent.py`、`prompt_template.py`、`pyproject.toml`、`README.md`、`test.py`、`uv.lock` 和空的 `snake` 目录。

启动命令为：

```sh
uv run agent.py snake
```

`snake` 是 `agent.py` 接收的第一个参数，指定本次任务面向的项目目录；该参数不构成文件访问或命令执行的权限边界，具体实现见下文“目标项目目录不构成权限边界”。启动后输入与模拟阶段相同的任务，Agent 实际创建 HTML、CSS 和 JavaScript 文件，并运行出可操作、能够计分的贪吃蛇游戏。作者把这个结果视为一个简化版 Claude Code。

该演示采用同步返回：每次等待模型生成完整响应后再显示结果。作者认为流式返回的体验可能更好，但会增加代码复杂度，因此没有在这个最小实现中引入。

### 三个工具与一个主循环

入口代码把三个函数放入 `tools` 列表：

- `read_file`：读取文件；
- `write_to_file`：写入文件；
- `run_terminal_command`：运行终端命令。

随后创建 `ReActAgent`，传入工具列表、`openai/gpt-4o` 和项目目录，再把用户任务交给 `agent.run()`。`ReActAgent` 的 `run` 方法负责完整循环：

1. 通过 `render_system_prompt` 渲染系统提示词，把它和 `<question>` 包裹的用户任务组成消息列表。
2. 在 `while` 循环中调用 `call_model` 请求模型。
3. 从响应中提取 `<thought>`。
4. 如果响应包含 `<final_answer>`，提取答案并结束。
5. 否则解析 `<action>` 中的函数名和参数。
6. 执行对应工具，把结果包装为 Observation 后加入消息列表。
7. 回到循环开头，让模型根据工具结果决定下一步。

运行终端命令具有更高风险，因此代码在执行 `run_terminal_command` 前主动询问用户是否继续。这个确认只覆盖演示中的一类危险操作，不代表该最小实现已经具备完整的权限治理、沙箱、回滚或结果验证能力。

### Agent 主程序是中介

从时序关系看，完整流程包含用户、Agent 主程序、模型和工具四个角色。用户任务先进入 Agent 主程序；主程序请求模型并展示 Thought 与 Action；随后执行 Action 指定的工具，把工具结果展示给用户并加入历史消息。模型读取新的 Observation 后继续判断，直到返回 Thought 与 Final Answer。

因此，工具只是函数，模型只生成行动请求，Agent 主程序才是连接模型、工具、消息历史和用户的运行中介。

## Plan-And-Execute：先规划，再动态调整

ReAct 不是唯一的 Agent 构建模式。很多 Agent 会先建立待办列表，再按计划执行；这类“先规划、再执行”的实现并没有统一名称，不同产品的具体流程也不完全相同。本文介绍的是 LangChain 提出的 Plan-And-Execute 模式。

Plan-And-Execute 不只生成一次静态计划，还会根据每一步的实际结果动态调整后续计划。它由四个部分组成：

- **Plan 模型**：根据用户问题生成初始执行计划；
- **Re-Plan 模型**：根据执行结果修改计划，或在任务完成时返回最终答案；
- **执行 Agent**：完成计划中的当前步骤；
- **Agent 主程序**：传递问题、计划、执行记录和最终答案，串联整个流程。

Plan 模型与 Re-Plan 模型可以使用同一个模型，也可以分别使用两个模型。执行 Agent 内部可以采用 ReAct，也可以采用其他运行模式；Plan-And-Execute 只要求它能够完成指定步骤。

### 三步查询示例

示例问题是“今年澳网男子冠军的家乡是哪里”。Plan 模型生成三步计划：

1. 查询当前日期；
2. 根据当前年份查询澳网男子冠军的名字；
3. 根据冠军名字查询其家乡。

主程序先把第一步交给带有网络搜索工具的执行 Agent。得到当前日期后，它把用户问题、原计划和执行记录一起发送给 Re-Plan 模型。已经完成的“查询当前日期”从新计划中移除，原本含糊的“查询对应年份冠军”被具体化为查询已经确定年份的冠军。

第二轮和第三轮采用相同结构：执行最新计划的第一步，把结果加入历史记录，再请求 Re-Plan 模型。最后一步完成后，Re-Plan 模型发现问题已经可以回答，不再返回新计划，而是直接返回最终答案。主程序把答案转交给用户，流程结束。

因此，Re-Plan 模型存在两种合法输出：新的执行计划，或者最终答案。把它描述成只负责返回新计划并不准确。

## 两种模式的关系

ReAct 把推理、工具请求和观察结果组织为逐步循环，适合让模型根据每次反馈决定下一步。Plan-And-Execute 在循环外增加显式计划，并在每一步之后重新规划；它的执行 Agent 仍然可以使用 ReAct。

两种模式都依赖 Agent 主程序保存必要历史、执行工具并判断模型输出属于工具请求还是最终答案。系统提示词能够规定交互协议，但不能替代实际的工具执行、风险确认和外部结果检查。

## 实践入口与运行前提（2026-09-22 补充核验）

本节是知识库维护时依据作者公开源码与官方教程补充的复现说明，不是视频原话，也不表示已完成模型调用或游戏验收。本版保留实践补充与权限澄清，并补齐空目录情况下的目标路径输入步骤；2025-07-22 是视频发布日期，2026-09-22 是补充与澄清的核验日期。

### 作者源码与固定版本

作者在视频的置顶评论中给出了 [VideoCode 仓库](https://github.com/MarkTechStation/VideoCode) 和 LangChain 教程入口。评论编号为 `268423268897`，发布者 ID `1815948385` 与视频作者一致；可通过[官方评论接口](https://api.bilibili.com/x/v2/reply?type=1&oid=114894200380730&sort=2)定位。

本次固定使用作者提交 `27052e6db5b91d5f65e8de008f37a090471c77a1`（2025-07-22 10:44:11，UTC+8，提交说明“添加.env相关的内容”）下的 [Agent的概念、原理与构建模式目录](https://github.com/MarkTechStation/VideoCode/tree/27052e6db5b91d5f65e8de008f37a090471c77a1/Agent的概念、原理与构建模式)。它是当日发布的源码版本，不声称与视频录制时的本地目录完全相同。该目录在本次取得的仓库最新版本中没有后续差异。

| 运行项 | 核实结果 | 依据 |
| --- | --- | --- |
| Python | 项目要求 `>=3.12`；`.python-version` 指定 `3.12` | 固定提交的 `pyproject.toml`、`.python-version` |
| 依赖 | 声明 `click>=8.2.1`、`openai>=1.91.0`、`python-dotenv>=1.1.1`；锁文件对应版本为 `8.2.1`、`1.91.0`、`1.1.1` | 固定提交的 `pyproject.toml`、`uv.lock` |
| 模型服务 | OpenAI SDK 请求 `https://openrouter.ai/api/v1`，模型名为 `openai/gpt-4o` | `agent.py` 的 `ReActAgent.__init__`、`main` |
| 凭据 | `load_dotenv()` 加载 `.env`，代码读取 `OPENROUTER_API_KEY`；缺失时抛出错误 | `README.md`、`get_api_key` |
| 项目目录 | 第一个参数必须是已存在的目录；入口使用 Click 的 `exists=True` 校验 | `main` 的参数声明 |
| 交互 | 启动后输入任务；仅 `run_terminal_command` 询问是否继续，输入 `Y` 或 `y` 才执行 | `run` |

### 按该版本准备并启动

先安装 Git、uv，并准备能够访问对应模型的 OpenRouter API Key。下面命令用于 macOS／Linux 终端；它们是基于源码整理的操作步骤，本轮没有执行模型请求。

```sh
git clone https://github.com/MarkTechStation/VideoCode.git
cd VideoCode
git checkout 27052e6db5b91d5f65e8de008f37a090471c77a1
cd 'Agent的概念、原理与构建模式'
uv python install 3.12
mkdir -p snake
```

在此目录新建 `.env`，写入下列内容，将占位文字替换为自己的 OpenRouter API Key，不要提交密钥文件：

```dotenv
OPENROUTER_API_KEY=替换为自己的密钥
```

先在上述项目目录的终端执行以下命令，复制输出的 `snake` 绝对路径。圆括号使目录切换只发生在子 Shell 中，执行后当前终端仍留在包含 `agent.py` 的目录：

```sh
(cd snake && pwd -P)
```

固定源码的 `render_system_prompt` 只向模板传入操作系统、工具列表和目录内文件列表，没有单独传入项目目录；新建空 `snake` 时，文件列表为空。因此，启动参数虽然指定了目录，模型的初始提示词仍无法从该列表得知目标路径。依据为上述固定提交的 `agent.py` 第 81—92 行及同目录的 `prompt_template.py` 环境信息段。

然后安装锁文件中的依赖并启动：

```sh
uv sync --locked
uv run --locked agent.py snake
```

程序显示“请输入任务：”后，输入下面这一行任务，先将其中的 `【粘贴上一步输出的绝对路径】` 完整替换为刚才取得的实际路径；不要把占位文字原样提交：

```text
请使用 HTML、CSS 和 JavaScript 创建可操作、能够计分的贪吃蛇游戏。目标目录为“【粘贴上一步输出的绝对路径】”。请将 HTML、CSS 和 JavaScript 分别保存到该目录中，调用文件工具时使用此目录下的绝对路径；如果需要运行命令，请明确使用该目标目录，不要假定终端当前目录已经切换到这里。
```

这是根据源码补充的任务输入示例，不是视频原话，也不保证模型一定遵守。它补足输出位置的上下文，不改变文件工具的访问范围、终端工作目录或操作系统权限；权限边界仍按下节说明理解。

作者 README 的原始启动命令是 `uv run agent.py snake`；这里加上 `--locked`，用于防止执行时静默更新依赖锁文件。`snake` 需自行创建：固定提交仅包含 `.python-version`、`README.md`、`agent.py`、`prompt_template.py`、`pyproject.toml`、`uv.lock` 六个文件，视频目录中的 `test.py` 和空 `snake` 目录没有发布到仓库。

`pyproject.toml` 还声明了未随仓库发布的 `snake3` 工作区成员。本轮用 uv `0.11.8` 执行 `uv lock --check --offline --python 3.14`，锁文件检查通过，未因该成员缺失失败；因此不把它描述成必现启动故障，也不要求读者擅改上游配置。默认 Python 3.12 的离线检查因本机没有该解释器而停止。上述检查只验证依赖元数据，不等同于 Python 3.12 环境下的安装、启动或模型调用成功。

若更换模型服务，须同时核对 `base_url`、密钥读取方式与模型标识；不能把 OpenRouter 的配置直接当作其他提供商配置。模型是否仍可调用以及账户权限，需要运行者在实际服务端核实。

### 目标项目目录不构成权限边界

本段依据固定提交 `27052e6db5b91d5f65e8de008f37a090471c77a1` 的 [agent.py](https://github.com/MarkTechStation/VideoCode/blob/27052e6db5b91d5f65e8de008f37a090471c77a1/Agent的概念、原理与构建模式/agent.py) 核对，日期为 2026-09-22。它澄清的是该示例的代码行为，不代表已完成运行测试。

`snake` 表示任务面向的项目目录，不能理解成“Agent 只被允许访问这里”。代码中的职责分别为：

| 位置 | 实际行为 | 不提供的保证 |
| --- | --- | --- |
| `main`（第 212—219 行） | Click 检查参数是已存在的目录，`os.path.abspath` 转换成绝对路径后传入 Agent | 不建立文件访问白名单或沙箱 |
| `render_system_prompt`（第 81—92 行） | 用 `os.listdir` 列出该目录下的条目，将条目的绝对路径写入提示词 | 提示词中的文件列表不限制工具接收的路径 |
| `read_file`、`write_to_file`（第 195—204 行） | 直接对模型传入的 `file_path` 调用 `open`；写入采用 `"w"` 模式 | 没有检查路径是否位于项目目录内，也没有逐次写入确认；已有文件可被覆盖 |
| `run_terminal_command`（第 206—210 行） | 使用 `subprocess.run(command, shell=True, ...)` 执行命令，未传入 `cwd`；入口也未调用 `os.chdir` | 不会自动切换到 `snake`，也不会把命令的访问范围限制在其中 |
| `run`（第 55—64 行） | 仅在工具名是 `run_terminal_command` 时询问是否继续 | 命令确认不等于目录隔离，文件读写工具不经过该确认 |

因此，模型提出项目目录以外的路径时，示例本身没有相应的目录越界拦截；操作是否成功仍受运行进程的操作系统权限等外部限制。这不意味着进程拥有无限权限。若按前节命令启动，未主动切换目录的终端命令继承的是启动 Agent 时的工作目录，而非参数指定的 `snake`。

### Plan-And-Execute 的官方实现与版本边界

作者置顶评论的[原始教程入口](https://langchain-ai.github.io/langgraph/tutorials/plan-and-execute/plan-and-execute/)在本次查询时返回页面跳转，目标是[当前 To-do list 中间件文档](https://docs.langchain.com/oss/python/langchain/middleware/built-in#to-do-list)。该目标不能当作视频当时的 Plan／执行／Re-Plan 示例。

阅读原流程请使用[历史版官方 Notebook](https://github.com/langchain-ai/langgraph/blob/aa1bbe3d01f98e862b7badee3e4b12881b807f62/docs/docs/tutorials/plan-and-execute/plan-and-execute.ipynb)。这里选取视频发布前的仓库提交 `aa1bbe3d01f98e862b7badee3e4b12881b807f62`（2025-07-22 00:41:55 UTC），用于固定当时可取得的教程；这不是对视频画面所用提交的认定。

该 Notebook 的运行前提与作者的 ReAct 项目不同：

- 通过 Notebook 安装 `langgraph`、`langchain-community`、`langchain-openai`、`tavily-python`，原命令使用 `-U`，没有固定这四个包的完整版本组合。
- 需要 `OPENAI_API_KEY` 与 `TAVILY_API_KEY`，分别用于模型与搜索；不能只配置前一个项目的 `OPENROUTER_API_KEY`。
- 执行 Agent 使用 `gpt-4-turbo-preview`，规划与重规划使用 `gpt-4o`；这是历史模型配置，本轮未核实其当前服务可用性。
- 教程明确要求 Pydantic v2 的 `BaseModel` 搭配 `langchain-core >= 0.3`，否则存在 v1／v2 混用错误。
- 示例使用 `StateGraph` 连接 `planner`、`agent`、`replan`，以 `Response`／`Plan` 区分最终回答与继续规划；调用采用 `app.astream`，设置 `recursion_limit=50`。

固定 Notebook 提交只能固定示例文本，不能自动固定依赖环境。它提供可追溯的历史实现入口；完整运行仍需建立兼容环境并验证服务权限，不承诺直接安装最新依赖即可复现。

## 来源与版本

| 编号 | 标题 | 作者 | 发布日期 | URL | 支持范围 |
| --- | --- | --- | --- | --- | --- |
| 原始资料 | Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code | 马克的技术工作坊 | 2025-07-22 | [原始链接](https://www.bilibili.com/video/BV1TSg7zuEqR/) | 本文原始转述来源；代码名称与流程以视频当次演示为准 |

版本关系：本版承接 [[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 权限澄清版）|权限澄清版]]，补充获取目标绝对路径和输入任务的步骤；此前初版、实践补充版及权限澄清版均完整保留。
