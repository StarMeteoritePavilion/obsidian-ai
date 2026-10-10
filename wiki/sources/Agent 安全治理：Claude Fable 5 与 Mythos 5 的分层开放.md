---
title: Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放
source: https://www.bilibili.com/video/BV1yyjN6vEZz
author: 唐国梁Tommy
published: 2026-06-15
ingested: 2026-09-04
updated: 2026-10-10
tags:
- AI
- 模型原理
- 大语言模型
- Claude
- Agent
- AI安全
- Anthropic
- 资料摘要
---

# Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放

原始资料：[[raw/sources/应用工程/AI Agent/刚发布几天就被美国政府紧急叫停：Claude 最强模型 Fable 5 ／ Mythos 到底强在哪？|Claude Fable 5 与 Mythos 5：模型能力与服务安全分析 (2026年6月)]]

## 核心结论

资料把 Claude Mythos 5 与 Fable 5 描述为同一高能力底座的两种开放形态。Mythos 5 面向经过审查的网络安全、关键基础设施与 Project Glasswing 合作方；Fable 5 面向普通用户和开发者，在同一底座外增加安全分类器、拒绝、自动回退和访问控制。

真正的能力对象是“模型＋Agent 执行框架”：模型负责推理、代码理解、长上下文与工具决策，外部系统负责命令、编辑、测试、浏览器、调试、权限、日志和验证。资料将这一组合称为 Agentic Execution，而不是把工具执行归因于模型参数本身。

## 规格与访问方式

现存原文记录的 2026 年 6 月 9 日发布报价为每百万输入 Token 10 美元、每百万输出 Token 50 美元。该历史报价不表示所有日期、渠道和服务条款下的最终账单；现存原文没有保留上下文容量、单次输出上限或唯一思考模式的规格断言，本摘要不继续引用这些旧表述。（[[raw/sources/应用工程/AI Agent/刚发布几天就被美国政府紧急叫停：Claude 最强模型 Fable 5 ／ Mythos 到底强在哪？#从模型能力转向 Agentic Execution|历史报价与运行环境]]）

## 网络安全门控

原文将 Mythos 5 的网络安全能力放在 Project Glasswing 的受控访问范围内，Fable 5 则增加分类器、拒绝与回退。N-day 指已公开的漏洞；发布补丁不意味着所有受影响系统已完成更新，资产盘点、补丁落地与运行时权限仍需单独治理。（[[raw/sources/应用工程/AI Agent/刚发布几天就被美国政府紧急叫停：Claude 最强模型 Fable 5 ／ Mythos 到底强在哪？#Project Glasswing 与网络安全边界|访问与防御边界]]）

资料记录的早期数据称，超过 95% 的 Fable 5 **会话（sessions）**不触发回退，分母不是一般请求。分类器触发时可拒绝或回退到 Opus 4.8；本摘要撤下现存原文已不再保留的漏洞利用率、利用数量、耗时、成本与越狱统计，不把未经原始评测核验的数字作为治理依据。（[[raw/sources/应用工程/AI Agent/刚发布几天就被美国政府紧急叫停：Claude 最强模型 Fable 5 ／ Mythos 到底强在哪？#同一底座的四层区别架构|分类器、回退与会话分母]]）

## 生命科学与工程价值

Mythos 5 在内部蛋白设计流程的 14 个靶点中产生 9 个值得继续研究的结果，某些药物设计环节约提速 10 倍。资料还称其分子生物学假设在盲评中约 80% 的情况下比 Opus 级模型更受科学家偏好。这些结果来自厂商材料，不等于独立复现或已经验证的临床收益，也不能外推为其他实验室或任务的固定收益。（[[raw/sources/应用工程/AI Agent/刚发布几天就被美国政府紧急叫停：Claude 最强模型 Fable 5 ／ Mythos 到底强在哪？#生命科学领域的收益与双重用途|生命科学案例与限制]]）

工程案例包括在约 5,000 万行 Ruby 代码库中用一天完成原本需要团队工作两个多月的迁移任务，以及长文档分析、图表读取、截图转网页和视觉游戏操作。高价值长任务适合调用该层级模型，普通任务则应通过模型路由使用更低成本模型。

## 治理边界

Agent 安全必须覆盖工具权限、生产环境访问、沙箱、审计、人工审批、独立验证、回滚与 Prompt Injection 检测。资料称 Fable 5 与 Mythos 5 要求 30 天数据保留，数据不用于训练新模型，而用于安全监测、攻击检测和减少误判；这是发布时政策，不是长期不变的接口保证。

现存原文已经补入厂商时间线：2026 年 6 月 9 日发布，6 月 12 日公告称收到美国政府出口管制指令并暂停访问，7 月 1 日发布页记录恢复访问。这些是原文记录的厂商声明，不是政府命令原件；没有取得政府命令全文，不能把厂商声明提升为对命令内容的独立核验，也不能继续写成原文没有日期或措施。（[[raw/sources/应用工程/AI Agent/刚发布几天就被美国政府紧急叫停：Claude 最强模型 Fable 5 ／ Mythos 到底强在哪？#发布与访问状态时间线|发布、暂停与恢复时间线]]、[[raw/sources/应用工程/AI Agent/刚发布几天就被美国政府紧急叫停：Claude 最强模型 Fable 5 ／ Mythos 到底强在哪？#来源与版本|来源归属]]）

待补政府命令全文及相关证据定位见 [[wiki/questions/原始资料待补证清单]]。

## 关联

- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- Agent 安全评估：[[wiki/sources/Agent 安全评估：AutoControl Arena 与对齐幻觉]]
- Claude Code Runtime：[[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]]
- 模型能力与外部执行边界：[[wiki/sources/大语言模型：Qwen 3.5 的 MoE、混合注意力与应用演示]]
