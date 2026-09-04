---
title: 驾驭工程：Claude Code Agent Runtime 架构拆解
source: https://www.bilibili.com/video/BV1zR9JBREua
author: 唐国梁Tommy
published: 2026-04-02
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - Agent
  - Agent Runtime
  - Claude Code
  - 驾驭工程
  - 资料摘要
---

# 驾驭工程：Claude Code Agent Runtime 架构拆解

原始资料：[[raw/sources/应用工程/驾驭工程/Claude Code源码曝光 底层技术硬核拆解：1884个文件背后，Anthropic如何构建Agent Runtime？|Claude Code 源码架构拆解]]

## 核心判断

资料把 Claude Code 定义为生产级 Agent Runtime，而非仅调用模型的命令行封装。作者分析的材料包含 1,884 个 TypeScript 源文件和 42 种以上工具能力；`main.tsx`、`claude.ts` 与 `query.ts` 分别承担 CLI 编排、API 客户端和 Agent 循环职责。

这一结论来自作者对其所称“网上流出的部分核心源码”的审计。视频与简介没有提供源码仓库、Commit、版本号或可复核快照，因此内部数量与行为只代表作者分析的材料，不能直接视为 Claude Code 其他版本的固定事实。

## 运行闭环与恢复

核心 ReAct 循环依次执行上下文准备、模型流式调用、工具执行、附件收集和终止／继续判断。模型提出工具调用后，系统先经过权限与 Hook，再通过流式或批量执行器运行；工具结果、任务通知、记忆和文件变更重新进入下一轮上下文。

413 上下文溢出会触发 Reactive Compact，输出截断时最大输出 Token 从 8K 提升到 64K，并最多恢复三次。更完整的七层恢复还覆盖 API 指数退避、529 过载、上下文排空、模型 Fallback 和无人值守持久重试，说明异常处理属于主循环，而非外围补丁。

## 六项实现取舍

- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 把可全局缓存的静态提示与记忆、MCP、环境等动态提示分开。
- Snip、Micro Compact、Auto Compact 与 Reactive Compact 形成四级压缩，并在压缩后恢复近期文件、Plan 与 Skill。
- Copy-on-Write Overlay 允许系统在用户确认前投机执行，确认后合并，拒绝后丢弃。
- BashTool 以 20 项检查和解释器黑名单约束 Shell 攻击面。
- 自研 Zustand 风格 Store 用 `Object.is`、Selector 和 `useSyncExternalStore` 控制终端 UI 更新。
- 六类 Hook 与 24 种事件把工具、权限、会话、压缩和用户输入暴露为扩展点。

这些取舍分别服务于成本、上下文连续性、交互延迟、安全、渲染性能和企业定制，没有合并成单一抽象。

## 多 Agent、记忆与提示编排

多 Agent 层包含 Fork、In-Process 和 Split-Pane 三种执行模型，以及按研究、综合、实现、验证组织的 Coordinator。`PlanAgent`、`ExploreAgent`、`VerificationAgent` 和 `ClaudeCodeGuide` 形成内置角色组，体现 Manager–Worker 与 Plan–Execute 分离。

记忆分为短期、工作、长期、摘要与 Checkpoint 五层。长期记忆由 `MEMORY.md` 索引 `user`、`feedback`、`project`、`reference` 四类主题文件，资料称每次最多选取五个相关文件；Checkpoint 通过 `--resume` 恢复会话与变更记录。

Prompt 工厂按 Override、Coordinator、Agent、Custom、Default、Append 六级组装提示。工具描述通过 API 的 `tools[]` 独立传递；工具超过 20 个时先经 Tool Search 发现，以控制一次性上下文负担。

## 安全与观测边界

安全链路由 Zod Schema、八源权限、两阶段权限分类、Hook、文件边界、Shell 检查和 Docker 沙箱组成；自动模式连续拒绝五次后回到交互模式。日志中的敏感内容还通过专门的类型标记限制误记录。

资料把可观测性列为成熟 Agent Runtime 的第五个维度，但只提到遥测、任务通知、文件变更和代码归因，没有单独解释采集与消费结构。该部分证据弱于多 Agent、记忆、Prompt 和安全四个维度。

## 架构边界

资料认为 Claude Code 不预建 Embedding 代码索引，也不做 AST 语法分析，而以模型推理配合 `grep`、`glob` 实时搜索理解代码库。这减少了预处理与索引维护，也可能在超大代码库中形成效率瓶颈。

该案例把 [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness|Agent Harness]] 的信息治理、行动接口、控制流、安全、观测和恢复职责落到了同一运行时。它说明生产 Agent 需要模型之外的完整执行结构，但不能在缺少源码版本和独立复核的情况下，把视频中的实现细节外推为长期稳定接口。

## 关联

- Harness 综合：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- 循环与恢复：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 可进化 Harness：[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness]]
