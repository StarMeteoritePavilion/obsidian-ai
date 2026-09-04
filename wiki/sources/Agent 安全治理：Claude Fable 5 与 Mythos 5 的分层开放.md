---
title: Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放
source: https://www.bilibili.com/video/BV1yyjN6vEZz
author: 唐国梁Tommy
published: 2026-06-15
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 应用工程
  - AI Agent
  - 安全治理
  - Claude
  - Anthropic
  - 资料摘要
---

# Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放

原始资料：[[raw/sources/应用工程/AI Agent/刚发布几天就被美国政府紧急叫停：Claude 最强模型 Fable 5 ／ Mythos 到底强在哪？|Claude Fable 5 与 Mythos 5 到底强在哪]]

## 核心结论

资料把 Claude Mythos 5 与 Fable 5 描述为同一高能力底座的两种开放形态。Mythos 5 面向经过审查的网络安全、关键基础设施与 Project Glasswing 合作方；Fable 5 面向普通用户和开发者，在同一底座外增加安全分类器、拒绝、自动回退和访问控制。

真正的能力对象是“模型＋Agent 执行框架”：模型负责推理、代码理解、长上下文与工具决策，外部系统负责命令、编辑、测试、浏览器、调试、权限、日志和验证。资料将这一组合称为 Agentic Execution，而不是把工具执行归因于模型参数本身。

## 规格与访问方式

两款模型支持 1,000,000 Token 默认上下文和最高 128,000 Token 单次输出，价格为每百万输入 Token 10 美元、每百万输出 Token 50 美元。Adaptive Thinking 是唯一思考模式，开发者通过思考力度控制深度，原始思维链不返回，只能隐藏或显示摘要。

资料所引材料没有公开参数量、训练数据规模或模型结构，因此这些公开规格不能用于推断底层架构。价格、数据保留和开放范围也属于 2026 年 6 月的产品口径，实际使用前需要重新核对。

## 网络安全门控

资料转述 Mythos Preview 在 OpenBSD 和 FFmpeg 中分别发现存在 27 年与 16 年的漏洞；Firefox 红队测试中，Opus 4.6 的漏洞利用成功率不到 1%，Mythos Preview 为 72.4%。针对补丁后的 N-day 漏洞，资料称模型在 18 个 Firefox 补丁中构建出 8 个远程代码执行利用程序，第一个耗时不到 1 小时；21 个 Windows 内核漏洞中生成 8 条本地提权链，平均每条 API 成本约 2,000 美元。

服务层门控把 Firefox 漏洞利用任务上的结果从 Mythos 5 的 88.4% 降到 Fable 5 的 0%；自动化越狱攻击成功率则从 Opus 4.6 的 83.2%、Opus 4.8 的 56.0% 降到 Fable 5 的 5.4%。超过 1,000 小时外部漏洞悬赏没有发现通用绕过方式，但这不构成不存在其他绕过路径的证明。

Fable 5 的分类器被有意设置得较保守，合法安全教学、授权测试和正常生物研究也可能被拒绝或回退到 Opus 4.8；资料同时称超过 95% 的一般请求不触发回退。安全因此是服务架构的取舍，而不是无成本的能力开关。

## 生命科学与工程价值

Mythos 5 在内部蛋白设计流程中处理 14 个靶点并产生 9 个有潜力的候选，某些药物设计环节约提速 10 倍。资料还称其分子生物学假设在盲评中约 80% 的情况下比 Opus 级模型更受科学家偏好。这些结果来自厂商材料，不能外推为其他实验室或任务的固定收益。

工程案例包括在约 5,000 万行 Ruby 代码库中用一天完成原本需要团队工作两个多月的迁移任务，以及长文档分析、图表读取、截图转网页和视觉游戏操作。高价值长任务适合调用该层级模型，普通任务则应通过模型路由使用更低成本模型。

## 治理边界

Agent 安全必须覆盖工具权限、生产环境访问、沙箱、审计、人工审批、独立验证、回滚与 Prompt Injection 检测。资料称 Fable 5 与 Mythos 5 要求 30 天数据保留，数据不用于训练新模型，而用于安全监测、攻击检测和减少误判；这是发布时政策，不是长期不变的接口保证。

官方视频标题写有“被美国政府紧急叫停”，但正文、简介和画面没有给出机构、命令、日期或措施，当前资料无法验证该说法，知识库不把它作为事实结论。

## 关联

- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- Agent 安全评估：[[wiki/sources/Agent 安全评估：AutoControl Arena 与对齐幻觉]]
- Claude Code Runtime：[[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]]
- 模型能力与外部执行边界：[[wiki/sources/大语言模型：Qwen 3.5 的 MoE、混合注意力与应用演示]]
