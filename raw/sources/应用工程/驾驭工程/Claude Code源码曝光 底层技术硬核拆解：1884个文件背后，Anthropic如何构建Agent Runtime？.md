---
title: "Claude Code源码曝光 底层技术硬核拆解：1884个文件背后，Anthropic如何构建Agent Runtime？"
source: https://www.bilibili.com/video/BV1zR9JBREua
author: 唐国梁Tommy
created: 2026-04-02
tags:
  - AI
  - Agent
  - Agent Runtime
  - Claude Code
  - 驾驭工程
---

> Claude Code Agent Runtime 独立专题

Claude Code 表面上是 Anthropic 官方推出的 AI 编程命令行工具，内部却包含模型调用、工具执行、上下文管理、权限控制、记忆、多 Agent 编排和恢复机制。按资料作者的定义，它不是把 API 结果打印到终端的简单封装，而是一套完整的 Agent Runtime。

这项分析来自作者对其所称“网上流出的部分核心源码”的架构审计。视频与简介没有提供源码仓库、Commit、版本号或可复核的代码快照，因此下文的文件数量、内部名称和行为均对应作者分析的那份材料，不能直接视为 Claude Code 其他版本的固定实现。

## 从命令行工具到 Agent Runtime

资料称，所分析的代码包含 1,884 个 TypeScript 源文件，并内置 42 种以上的工具调用能力。其中 `main.tsx` 接近 4,700 行，负责 CLI 总编排；`claude.ts` 超过 3,600 行，承担 API 客户端职责；`query.ts` 超过 1,700 行，包含 Agent 循环引擎。

系统使用 TypeScript 与 React Ink 渲染终端界面，并包含 Zustand 风格状态管理、多 Provider 适配、分层记忆、权限、安全沙箱、遥测和多 Agent 编排。这些模块共同把模型的文本输出转成可执行、可恢复并受约束的操作。

## ReAct 循环的五个阶段

`query.ts` 中的 `while (true)` 循环实现了经典 ReAct 模式：模型推理下一步行动，系统执行工具，把结果交回模型，再根据新状态继续判断。作者把一次循环拆成五个阶段。

### 上下文准备

每轮调用模型前，系统先裁剪过旧的历史消息，压缩已经缓存的工具结果，并在上下文过长时生成全量摘要。目标是在保留任务所需信息的同时控制窗口占用。

### 模型流式调用

对话历史、系统提示和可用工具被打包后发送给 Claude 模型。系统在流式返回过程中同时收集文本和工具调用意图，不必等待整段响应完成。

### 工具执行

工具调用有两条执行路径：流式执行器可以在模型仍在输出时提前开始任务，批量执行器则等待全部调用确定后统一执行。真正执行前还要经过权限检查与 Hook；权限不足的操作请求用户确认，Hook 可以拦截或修改行为。

### 附件收集

工具完成后，系统收集任务通知、记忆、文件变更记录等附件，为下一轮模型调用补充状态。

### 终止或继续

模型没有提出工具调用时，任务被视为完成；存在工具结果时，循环继续。若请求因上下文过长返回 413，系统触发 Reactive Compact 后重试；若输出被截断，最大输出 Token 从 8K 提升到 64K，最多恢复三次。

## 七层恢复策略

为了应对网络抖动、API 过载、上下文溢出和长时间无人值守，资料列出七层恢复策略：

1. API 指数退避重试。
2. 529 过载处理。
3. 输出 Token 恢复。
4. 响应式压缩。
5. 上下文排空。
6. 模型 Fallback。
7. 无人值守持久重试。

退避最长可达 5 分钟，重置上限为 6 小时。循环本身由 Async Generator 实现，消息和进度通过 `yield` 持续推送给界面，使恢复过程与流式交互使用同一条更新通道。

## 六项工程设计

### Prompt 缓存分割

系统用 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 把系统提示拆成静态段和动态段。身份、工具使用指南与编码原则等稳定内容使用全局缓存；记忆、MCP 指令和环境信息等变化内容不缓存。该分割利用 Anthropic API 的 Prompt Caching，提高不同会话复用静态前缀的机会。

### 四级上下文压缩

上下文压缩不是单一摘要动作，而是四级递进体系：

1. **Snip**：每轮调用前进行轻量裁剪。
2. **Micro Compact**：采用缓存感知、基于时间和 API 级压缩三种策略。
3. **Auto Compact**：Token 超过阈值后生成结构化全量摘要。
4. **Reactive Compact**：发生 413 上下文溢出后紧急压缩并重试。

压缩完成后，系统按优先级恢复最近读取的文件、当前 Plan 文件和已经调用的 Skill。它并非只删除旧消息，而是先缩减信息，再把当前任务最重要的材料带回上下文。

### Copy-on-Write 投机执行

在用户确认建议之前，系统可以先在 Copy-on-Write Overlay 文件系统中执行。写操作进入临时覆盖层；用户同意后再把结果复制回真实文件系统，拒绝后直接丢弃覆盖层。当前建议等待确认时，下一条建议也可以进入投机执行，以流水线方式隐藏等待延迟。

### BashTool 安全检查

资料称 BashTool 包含 20 项安全检查，覆盖不完整命令、Shell 元字符、嵌入换行、命令替换、IFS 注入、Token 注入、Unicode 空白伪装、I/O 重定向、`jq system()` 和解释器调用等攻击面。自动模式还设置了解释器黑名单，Python、Node、Ruby、Perl、PHP 与 Bash 等脚本入口默认不能无确认执行。

### Zustand 风格轻量 Store

Claude Code 没有直接采用 Zustand，而是实现了相似的轻量 Store。它使用 `Object.is` 做引用比较，通过 Selector 订阅只更新相关字段，并以稳定的 Store 实例配合 React Ink 的 `useSyncExternalStore` 控制终端重渲染。资料称 `AppState` 有 100 多个属性，覆盖设置、任务、工具、权限、MCP 状态和投机执行状态。

### Hook 系统

Hook 系统包含六种类型：`command` 执行 Shell 命令，`prompt` 让 LLM 审查，`agent` 启动多轮 Agent，`http` 调用外部端点，`callback` 调用内部 TypeScript 函数，`function` 执行简单布尔检查。

24 种事件覆盖工具调用、权限、会话、生命周期、数据压缩和用户输入等环节。企业可以在不修改核心代码的情况下，在命令执行前记录日志，或在写入代码前增加安全审查。

## 成熟 Agent Runtime 的五个维度

### 多 Agent 编排

资料列出三种 Agent 执行模型：

- **Fork 子 Agent**：作为隐式 Worker 继承父 Agent 的完整上下文，在独立分支执行。
- **In-Process**：在同一进程异步运行，并通过 `AsyncLocalStorage` 隔离上下文。
- **Split-Pane**：在 tmux 或 iTerm2 中打开分窗格，让 Leader 与 Teammate 分别运行。

Coordinator 模式把工作流划分为研究、综合、实现和验证四个阶段，再把子任务分配给不同 Worker。内置角色包括负责规划的 `PlanAgent`、探索代码库的 `ExploreAgent`、测试验证的 `VerificationAgent` 和提供使用指导的 `ClaudeCodeGuide`。各 Worker 可以拥有不同 Prompt、工具集和模型，形成 Manager–Worker 与 Plan–Execute 分离的层次化架构。

### 五层记忆

记忆体系覆盖五个层面：

1. **短期记忆**：当前会话的消息列表，随进程结束消失。
2. **工作记忆**：任务状态、投机执行状态和 Skill 调用跟踪等运行信息。
3. **长期记忆**：在 `~/.claude/projects/.../memory/` 中保存，由 `MEMORY.md` 索引主题文件；内容分为 `user`、`feedback`、`project` 和 `reference` 四类。资料称系统用 Claude Sonnet 判断相关性，每次最多加载五个文件。
4. **摘要记忆**：上下文过长时生成结构化摘要。
5. **Checkpoint**：持久化会话，通过 `--resume` 恢复消息历史、文件变更记录和代码归因信息。

### Prompt 编排

系统提示由多层 Prompt 工厂动态组装，并按 Override、Coordinator、Agent、Custom、Default、Append 六级确定优先顺序。默认交互、Plan、Proactive、Coordinator 和 SDK 非交互等场景使用不同组合。

工具描述不写入系统提示，而是通过 API 的 `tools[]` 独立传递，并按当前状态动态生成。工具超过 20 个时，模型先通过 Tool Search 发现工具，再调用所需工具，以减少一次性注入的描述。

### 安全纵深

安全链路从 Zod Schema 输入验证开始，继续经过八个来源的权限规则、两阶段权限分类、Hook 拦截、文件边界、Shell 检查和 Docker 沙箱。自动模式先由 Sonnet 快速判断，再用扩展思考深入分析；连续拒绝五次后自动回到交互模式，避免分类器反复阻塞。

日志层还使用类型标记保护代码内容和文件路径。资料展示的类型名为 `Do_Not_Log_This_Content_Or_You_Will_Be_Fired`，用显眼的类型约束提醒开发者不要误记敏感内容。

### 可观测性

资料把完整可观测性列为第五个维度，并提到遥测、任务通知、文件变更和代码归因信息，但没有像前四个维度一样单独展开其数据结构、采集范围和使用方式。因此，这一维度只能保留为作者的架构判断，不能据此补出更具体的观测机制。

## 总体评价与适用边界

作者用成本意识、韧性设计、安全纵深和架构前瞻性概括这套架构。Prompt 缓存与分层压缩面向 Token 成本；七层恢复、会话持久化和压缩后恢复面向长时运行；多层权限与命令检查面向真实文件和 Shell 风险；投机执行、Coordinator、KAIROS 与每日日志蒸馏等仍在特性门控后的能力，则体现了未来扩展方向。

代码库理解采用了另一项明确取舍：不预先建立 Embedding 代码索引，也不做 AST 语法分析，而是依赖模型推理与 `grep`、`glob` 实时搜索。作者认为这种“模型即理解引擎”的路线在模型能力增强时成立，同时指出超大代码库的效率可能成为瓶颈。

这份材料展示了 Agent Runtime 如何把上下文、模型调用、工具、安全、状态、记忆、恢复和多 Agent 编排接入同一条执行链。它能作为工程结构参考，但视频没有给出可独立复核的源码版本，部分能力也仍受特性门控；具体实现和数量不能脱离 2026 年 4 月 2 日发布的视频直接外推到其他 Claude Code 版本。
