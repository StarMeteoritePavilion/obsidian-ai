---
title: 驾驭工程：Prompt、Context 与 Harness 的边界
source: https://www.bilibili.com/video/BV1B6DiBbEf8
author: 隔壁的程序员老王
published: 2026-04-16
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 提示词工程
  - 上下文工程
  - Harness
  - 应用工程
  - 资料摘要
---

# 驾驭工程：Prompt、Context 与 Harness 的边界

原始资料：[[raw/sources/应用工程/驾驭工程/提示词工程 上下文工程 Harness工程 是什么？|提示词工程 上下文工程 Harness工程 是什么？]]

## 核心结论

资料把 Prompt Engineering 概括为解决“怎么问”，Context Engineering 概括为解决“怎么记”，Harness Engineering 概括为解决“怎么管”。三者描述不同的主要职责，但实现方式可以重叠：预制提示词能够规定 Harness 工作流，提示词本身也属于上下文，权限和代码测试率检查则位于模型调用之外。

## Prompt 与 Context

提示词可以加入角色设定或要求模型分步处理问题。资料用辽宁朝阳的东北话爱好者和分解算术题说明，改变输入方式可以改变模型表达与处理过程。

上下文则来自当前消息与历史记录的拼接。工具型任务连续访问十几个网页时，每页几十 K 或几百 K 的返回可能转换成上万 Token，逐渐淹没最初指令。资料列举删除 HTML、总结旧历史和直接删除大记录三类控制方法。

## Harness 的三个例子

资料用开发游戏说明 Harness 的流程约束：先明确需求，再写文档、划分任务、开发、测试、打包和发布。除此之外，只允许 AI 读写项目文件夹属于权限 Harness；监控代码测试率并在不足时要求补测属于质量 Harness。

这一口径比“模型之外的一切”更窄，重点落在开发者预设的行为规则、权限和检查。它与本库其他来源对 Harness 的定义并存，不提升为统一标准。

## 模型能力边界

作者记录，一套使用多种工程技巧、实际效果不错的 AI 系统，被某个未发布模型在不使用这些技巧时直接超过。由于厂商、模型、任务、评测方法和差距均未公开，该案例只能支持“工程收益需要与基础模型升级持续比较”，不能证明 Harness 必然失效。

作者用“小马、马鞍与汽车”表达这些技巧可能被模型能力跨代提升取代的判断，并把这种现象称为 The Bitter Lesson（苦涩的教训）。资料没有进一步讲解该概念，因此本页不补入外部定义。

## 定位与关联

官方“AI技术”合集将本文列为第 13 条。视频完整对照 Prompt、Context 与 Harness，但没有期数标识，也不属于知识库现有 Prompt、Context 或 Harness 课程系列的一期，因此定位为应用工程／驾驭工程独立对照专题。

- 提示词工程：[[wiki/syntheses/提示词工程：从单轮指令到生产规范]]
- 上下文工程：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Harness 综合：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- 既有 Prompt／Context 导论：[[wiki/sources/上下文工程：从 Prompt 到 Agent 上下文治理]]
- Harness 系列收束：[[wiki/sources/驾驭工程：系列完结，下一步该往哪走？]]
