---
title: 循环工程：先导篇 从 Prompt 到 Loop
source: https://www.bilibili.com/video/BV12MKp6oEKr
author: 晴天AI实战
published: 2026-07-20
ingested: 2026-07-25
tags:
  - AI
  - 循环工程
  - 应用工程
  - 资料摘要
---

# 循环工程：先导篇 从 Prompt 到 Loop

原始资料：[[raw/sources/应用工程/循环工程/先导篇：从 Prompt 到 Loop|先导篇：从 Prompt 到 Loop]]

## 核心结论

Loop Engineering 管理跨轮次决策流：系统自行生成下一步任务、执行、检查、反馈并决定何时停止。人的角色由循环内部逐轮提供 Prompt 的“人肉时钟”，转变为循环外部设计目标、边界、反馈和停止条件的调度者。

## 四层预告

- **Prompt**：表达当前任务。
- **Context**：管理当前轮次可见的信息。
- **Harness**：提供工具、执行环境、约束和安全边界。
- **Loop**：组织跨轮次的任务、执行、检查、反馈与停止。

这一结构在先导篇中用于预告系列路线。第一期进一步把结构精化为 Prompt、Context、Loop 三层递进，以及包住三层的 Harness 外壳。

## 系列路线

- 第一期展开 Prompt、Context、Loop 三层递进、Harness 外壳和人的位置迁移。
- 第二期比较三种循环设计及其模型信任边界。
- 第三期拆解循环组件，并讨论 297 美元交付 5 万美元合同的极端案例。
- 第四期说明循环如何复用上下文工程基础设施，并给出五步搭建路线。

## 风险边界

资料提出验证债、理解腐烂、认知投降和 Token 失控四类工程账目，但先导篇没有展开机制与解决方案。音轨中的术语为 Ralph，先导篇只预告相关极端案例，没有解释其实现。

## 后续资料修正

先导篇把 Prompt、Context、Harness 和 Loop 排成四层。第一期明确把这一画法修正为 Prompt、Context、Loop 三层递进，Harness 是包住三层并横切工具、权限、安全与恢复的工程外壳。两份资料均保留，综合页采用第一期的精化模型。（[[wiki/sources/循环工程：第一期 什么是 Loop Engineering？|什么是 Loop Engineering]]）

## 关联

- 基础设施边界：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 跨资料综合：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
