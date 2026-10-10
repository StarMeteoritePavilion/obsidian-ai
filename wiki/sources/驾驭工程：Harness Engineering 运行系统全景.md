---
title: 驾驭工程：Harness Engineering 运行系统全景
source: https://www.bilibili.com/video/BV1VBX9BrEon
author: 唐国梁Tommy
published: 2026-03-29
ingested: 2026-09-04
updated: 2026-10-10
tags:
- AI
- Agent
- Agent Harness
- Harness Engineering
- 驾驭工程
- 资料摘要
---

# 驾驭工程：Harness Engineering 运行系统全景

原始资料：[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime|为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime]]

## 核心结论

资料把 Harness Engineering 定义为模型外部运行系统的设计：它不只决定模型看到什么，还管理任务拆解、工具、权限、状态、验证、恢复、日志和人类接管。按照本资料的包含口径，Harness 包含 Context，Context 包含 Prompt；三者分别回答“怎样表达”“让模型看到什么”和“让模型在什么机制中工作并确保完成”。每步成功率 95%、连续 20 步约 35.8% 的例子明确假设各步骤完全独立（`0.95^20 ≈ 35.8%`），不是实测系统可靠性。（[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime#二、兴起原因|原文：独立步骤假设]]）

Harness 的价值不等于堆叠更多组件。Vercel 的厂商案例描述，删减冗余工具可减少步骤、Token 和响应时间并提高成功率。模型升级时应复测补偿组件：移除后是否仍达标、中间 Agent 是否增加延迟，以判断哪些机制可以删除。（[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime#三、跨组织案例对比与共性|原文：组织案例]]、[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime#五、风险与局限|原文：升级回归]]）

## 实践中的共同模式

原文将 Anthropic 案例概括为从双角色向 Planner、Generator、Evaluator 演进，并用进度文件与 Git 历史接续任务；Google DeepMind 的相关验证架构采用 Generator、Verifier、Reviser 循环。两类案例共同强调生成与评估分离，原文未提供足以独立复核各实现与收益的一手材料。（[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime#三、跨组织案例对比与共性|原文：角色与状态]]）

OpenAI 案例连接动态可观测数据、Linter、结构测试与后台清理；Stripe Minions 使用隔离且预热的 Devbox，通过 MCP 访问内部工具；Manus 案例强调生产就绪前的多次重写。这些案例用于说明机制，不证明通用收益。（[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime#三、跨组织案例对比与共性|原文：其他组织案例]]）

## 六个模块

资料将成熟 Harness 归纳为六部分：上下文与知识管理、工具编排与权限、验证与硬约束、状态与持续记忆、可观测性与反馈闭环、人类接管与生命周期管理。`AGENTS.md`、子 Agent 上下文隔离、Linter、结构测试、Pre-commit Hook、进度文件、检查点和审批点分别落在这些模块中。

三类控制必须分别承担责任：确定性约束拥有安全否决权，模型评判检查语义质量并提供重试反馈，人类接管处理高价值决策和超出预定义恢复逻辑的风险。模型评判不能替代系统级权限边界。（[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime#三类控制的责任边界（加深建议）|原文：三类控制]]）

## 证据与风险边界

三类主要风险是：模型升级会让补偿性组件变成过度工程；复杂搜索与多 Agent 配置可能放大风险；主要证据来自有利益关系的工具厂商，OpenAI 百万行代码案例也缺少普通工程团队的可复现证明。

BrowseComp 类评测的定性转述表明，多 Agent 配置可能产生更多非预期解法；复杂 Harness 或多 Agent 本身不证明风险更低。

组织案例的具体版本、实现和实验条件没有可独立定位的完整一手证据。（[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime#五、风险与局限|原文：证据边界]]、[[wiki/questions/原始资料待补证清单|待补证清单]]）

## 定位与关联

本文为应用工程／驾驭工程独立全景专题，没有期数标识。

- Harness 综合：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- Prompt、Context 与 Harness 边界：[[wiki/sources/上下文工程：提示词、上下文与 Harness 的职责边界]]
- Claude Code 运行时实例：[[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]]
- 可进化 Harness：[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness]]
- 循环工程：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- 评估工程：[[wiki/syntheses/评估工程：从通用基准到业务质量门]]
