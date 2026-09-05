---
title: 循环工程：第四期 搭好 Loop ≠ 能上线？
source: https://www.bilibili.com/video/BV11i3w61EKQ
author: 晴天AI实战
published: 2026-07-27
ingested: 2026-07-27
updated: 2026-08-12
tags:
  - AI
  - 循环工程
  - 应用工程
  - 资料摘要
---

# 循环工程：第四期 搭好 Loop ≠ 能上线？

原始资料：[[raw/sources/应用工程/循环工程/第四期：搭好 Loop ≠ 能上线？|第四期：搭好 Loop ≠ 能上线？]]

## 核心结论

Loop Engineering 是建立在 Context Engineering 基础设施之上的更高层结构。Context 管理每一轮的信息，Loop 复用相同的 Skill、隔离窗口、工具与外部连接，把单次执行扩展为能够触发、发现、记忆、评判和并行的跨轮流程。

能够运行不等于能够上线。发现源和状态文件只保证循环转起来；独立评判器、工作区隔离、Token 上限和人工复核才限制它运行后的风险。

## 共用基础设施

- **Agent Skills**：在 Context 层负责渐进式披露，在 Loop 层负责固化项目规则并减少意图债。
- **Sub-agents**：在 Context 层通过独立窗口探索并返回摘要，在 Loop 层分离 Maker 与 Checker，形成独立评判机制。
- **工具平台**：Context 侧承载检索、压缩、结构化笔记和 Prompt Caching，Loop 侧承载调度、完成条件、并行隔离和外部连接。

三层关系保持不变：Prompt 管任务表达，Context 管当前窗口，Loop 管跨轮决策，Harness 包住三层并提供共同的工程环境。

## 五步路线图

1. **触发**：先运行一个定时或自主节奏的循环。
2. **发现**：读取 CI、Issue、Commit 和待处理收件箱，自动分诊问题。
3. **状态**：把未完成事项写入仓库中的 Markdown 文件，供下一轮继续。
4. **评判**：使用 Fresh 模型根据明确完成条件作出独立判断。
5. **隔离**：让并行 Agent 各自在独立 Git worktree 中工作。

资料以发布时的 Claude Code `/loop`、`/goal` 和 `--worktree` 为例，并把 Codex Automations 与后台 Worktree 列为对应能力。具体命令属于 2026-07-27 的工具状态，不应脱离时间直接外推。

## 上线检查

上线前必须确认六项：

1. 循环有明确的发现源。
2. 跨轮状态保存在仓库文件中。
3. 存在能够拒绝结果的独立 Evaluator。
4. 并行任务使用隔离 Worktree。
5. 设置 Token 或预算上限。
6. 明确必须由人复核的步骤。

前两项决定循环能否运行，后四项决定运行后能否被约束。宁可先上线一个范围较小的循环，也不能省略评判器和人工复核点。

## 系列收束

本期资料明确说明它是 Loop Engineering 系列最后一期。系列形成的完整结构是：Prompt、Context、Loop 三层递进，Harness 作为共同外壳；下一阶段转向评估工程，通过定义质量、指标和评测集为优化提供方向。后续见[[wiki/sources/评估工程：第一期 排行榜遥遥领先，用起来怎么各种拉胯？|评估工程第一期]]。

## 关联

- 上一期：[[wiki/sources/循环工程：第三期 Loop 该怎么搭？]]
- 下一系列：[[wiki/sources/评估工程：第一期 排行榜遥遥领先，用起来怎么各种拉胯？]]
- 跨资料综合：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- 信息治理基础：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
