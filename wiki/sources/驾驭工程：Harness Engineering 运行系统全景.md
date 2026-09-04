---
title: 驾驭工程：Harness Engineering 运行系统全景
source: https://www.bilibili.com/video/BV1VBX9BrEon
author: 唐国梁Tommy
published: 2026-03-29
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - Agent
  - Agent Harness
  - Harness Engineering
  - 驾驭工程
  - 资料摘要
---

# 驾驭工程：Harness Engineering 运行系统全景

原始资料：[[raw/sources/应用工程/驾驭工程/为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime|为什么你的Agent总翻车？Harness Engineering全拆解]]

## 核心结论

资料把 Harness Engineering 定义为模型外部运行系统的设计：它不只决定模型看到什么，还管理任务拆解、工具、权限、状态、验证、恢复、日志和人类接管。按照本资料的包含口径，Harness 包含 Context，Context 包含 Prompt；三者分别回答“怎样表达”“让模型看到什么”和“让模型在什么机制中工作并确保完成”。

Harness 的价值不等于堆叠更多组件。Vercel 移除 80% 工具后反而减少步骤和 Token 并提高成功率；Anthropic 针对 Sonnet 4.5 设置的上下文重置，在 Opus 4.5 消除对应行为后失去必要性。资料据此强调，Harness 应当约束 Agent 的解空间，同时保持可删减、可替换并匹配当前模型能力。

## 实践中的共同模式

Anthropic 用初始化 Agent、编码 Agent、进度文件、Git 历史和 JSON 功能清单承接跨窗口任务，随后演进为 Planner、Generator、Evaluator 三角色结构。Google DeepMind 的 Aletheia 使用 Generator、Verifier、Reviser 循环。两套方案共同指向生成与评估分离：外部严格评估器比依赖生成器自我批评更容易工程化。

OpenAI 案例把代码库知识、确定性架构约束和周期性“垃圾回收”连接起来；Stripe Minions 使用隔离、预热的 Devbox，并通过 MCP 接入 400 多个内部工具；Manus 在六个月内经历五次完整重写。这些案例分别强调版本化制品、硬约束、执行环境和持续迭代，但均来自特定组织与条件，不能直接外推为通用收益。

## 六个模块

资料将成熟 Harness 归纳为六部分：上下文与知识管理、工具编排与权限、验证与硬约束、状态与持续记忆、可观测性与反馈闭环、人类接管与生命周期管理。`AGENTS.md`／`CLAUDE.md`、子 Agent 上下文隔离、Linter、结构测试、Pre-commit Hook、进度文件、检查点和审批点分别落在这些模块中。

这套结构重组了 Test Harness、DevOps、分布式系统、安全工程和 SRE 的既有实践，同时把约束对象从确定性程序扩展为概率性推理系统。其创新主要在组合方式、生成—评估分离、约束解空间和面向 Agent 可推理性的代码库设计，不代表所有组成都是新技术。

## 证据与风险边界

资料列出四类风险：Harness 概念可能膨胀为无所不包的术语；模型升级会让补偿性组件变成过度工程；主要案例来自工具厂商，缺少独立、定量、可复现的 Benchmark；OpenAI 百万行代码案例的空仓库、自有 Codex 与专家团队条件尚未在普通团队复现。

Harness 还可能放大风险。资料转述 Anthropic BrowseComp 复盘：非预期解法发生率在单 Agent 配置中为 0.24%，在多 Agent 配置中升至 0.87%。该数字只适用于对应配置与评测，不证明多 Agent 普遍更危险。

资料关于 Aletheia 的 95.1% 准确率、四个 Erdős 猜想数据库开放问题和更广泛问题集 68.5% 错误率也来自不同评测口径，只能说明专用系统的场景内成绩与场景外泛化可能同时存在明显差异，不能合并为统一能力结论。

## 定位与关联

官方“前沿论文与最新技术趋势洞察”合集将本文列为第 16 条，但官方标题、开场和结尾均没有期数标识，官方章节为空；正文完整介绍 Harness 的定义、行业案例、模块、旧技术重组和风险，因此定位为应用工程／驾驭工程独立全景专题，不把合集位置当作文章期数。

- Harness 综合：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- Prompt、Context 与 Harness 边界：[[wiki/sources/驾驭工程：Prompt、Context 与 Harness 的边界]]
- Claude Code 运行时实例：[[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]]
- 可进化 Harness：[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness]]
- 循环工程：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- 评估工程：[[wiki/syntheses/评估工程：从通用基准到业务质量门]]
