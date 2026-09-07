---
title: 循环工程：从 Prompt 到可持续 Loop
source:
  - https://www.bilibili.com/video/BV12MKp6oEKr
  - https://www.bilibili.com/video/BV1UnKh6QE5D
author:
  - 晴天AI实战
published: 2026-07-20
ingested: 2026-07-25
updated: 2026-09-07
tags:
  - AI
  - 循环工程
  - 应用工程
  - 资料摘要
---

# 循环工程：从 Prompt 到可持续 Loop

原始资料：[[raw/sources/应用工程/循环工程/循环工程入门：从 Prompt 到可持续运行的 Loop|循环工程入门：从 Prompt 到可持续运行的 Loop]]

## 核心结论

Loop Engineering 把逐轮触发、分工、反馈和推进交给系统。人的职责从循环内操作转为循环外设计目标、验证、停止、恢复和成本边界。

## 三层与外壳

- Prompt 管当前任务怎样表达。
- Context 管当前窗口包含什么信息。
- Loop 管系统怎样自行触发、分工、回写状态并持续运行。
- Harness 横切三层，提供工具、权限、安全和恢复；这一口径是第一期对先导篇“四层排列”的修正。

## 循环成立条件

循环至少需要触发、任务执行、结果验证、状态回写和停止判断。子 Agent 是一种分工手段，不是循环成立的必要产品；定时器也只解决唤醒，不证明任务完成。Loop会同时放大正确理解和错误状态，因此权限越大、运行越久，越需要独立验证和可恢复检查点。

## 关联

- [[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- [[wiki/sources/循环工程：第二期 三大流派与四笔代价]]
- [[wiki/sources/循环工程：组件、搭建与上线检查]]
