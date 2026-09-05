---
title: 循环工程：第三期 Loop 该怎么搭？
source: https://www.bilibili.com/video/BV1GB3g67EnG
author: 晴天AI实战
published: 2026-07-25
ingested: 2026-07-25
updated: 2026-08-12
tags:
  - AI
  - 循环工程
  - 应用工程
  - 资料摘要
---

# 循环工程：第三期 Loop 该怎么搭？

原始资料：[[raw/sources/应用工程/循环工程/第三期：Loop 该怎么搭？|第三期：Loop 该怎么搭？]]

## 核心结论

循环不是多次调用 Agent，而是一条由发现、交付、验证、持久化和调度组成的闭环。Automations、Worktrees、Skills、Connectors、Sub-agents 和 Memory 六个组件分别承担触发、隔离、规则复用、外部交付、独立验证和跨轮状态保存。

## 五个动作与六个组件

1. **发现**：Automation 调用 triage Skill，读取 CI 失败、开放 Issue 和最近 Commit；`SKILL.md` 保存稳定项目规则，减少反复交代产生的意图债。
2. **交付**：每个发现进入独立 Git worktree，由子 Agent 在隔离环境中起草修改。
3. **验证**：另一个子 Agent 使用不同指令或模型，依据 Skill、测试和行为证据独立审查。
4. **持久化**：Connector 创建 PR、更新 Ticket，Memory 保存状态；无法自动处理的事项进入人工待办。
5. **调度**：Automation 定时启动下一轮，使整条链从一次 Harness 执行变成真正的循环。

六个组件不是独立工具清单，而是对五个动作的支撑关系：Automations 负责调度，Worktrees 负责隔离交付，Skills 负责发现规则，Connectors 与 Memory 共同负责持久化，Sub-agents 负责独立验证。

## 独立评判器

资料引用 Anthropic 工程师 Prithvi Rajasekaran 的观察：生成者在原有上下文中评价自己的代码，容易沿着既有推导再次说服自己。让同一生成者“更挑剔”不如把生成和判断分开。

独立评判器应使用新上下文和不同指令，并通过 Playwright MCP 等工具实际操作页面、检查 DOM 和保留截图。判断依据由生成者的意图转为外部可观察的行为证据。

## Ralph 的 Backpressure

Ralph 每轮只做一件事，由主线程调度、子 Agent 执行重任务，并重新加载规格与 `fix_plan.md`。它使用三层 Backpressure：

1. 单元测试。
2. Rust 类型系统与编译检查。
3. 动态语言的静态分析和类型检查。

模型搜索既有实现不稳定时，可能因没有搜到结果而重复造轮子。资料的处理方式是在 Prompt 中明确要求先搜索，并根据反复出现的失败模式持续 Tuning。

## 案例边界

资料再次引用 297 美元 API 成本完成 5 万美元合同，以及构建 CURSED 编译器、LLVM 后端和标准库的案例。该数字属于个案；Ralph 的适用范围限于从零启动的绿地项目，不用于已有代码库。

## 关联

- 上一期：[[wiki/sources/循环工程：第二期 三大流派与四笔代价]]
- 下一期：[[wiki/sources/循环工程：第四期 搭好 Loop ≠ 能上线？]]
- 跨资料综合：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- 信息治理基础：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
