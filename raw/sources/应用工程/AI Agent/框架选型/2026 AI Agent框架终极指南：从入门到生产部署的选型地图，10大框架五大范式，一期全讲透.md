---
title: 2026 AI Agent框架终极指南：从入门到生产部署的选型地图，10大框架五大范式，一期全讲透
source: https://www.bilibili.com/video/BV1SqAVzLEci
author: 唐国梁Tommy
created: 2026-03-19
tags:
  - AI
  - AI Agent
  - 框架选型
---

> AI Agent 框架选型独立专题

2024 年，GitHub 上超过 1000 Stars 的 Agent 相关仓库从 14 个增至 89 个，增幅为 535%。本篇把这段时期称为 Agent 框架的“寒武纪大爆发”，并认为到 2026 年 3 月，竞争已经进入生态位逐渐清晰的阶段。

选型的关键不是寻找统一意义上的“最好框架”，而是理解各框架的架构范式、控制边界和部署约束，再选择最适合当前团队与场景的方案。本文所有生态、版本、Star 数、融资和趋势判断均对应 2026 年 3 月这一时间截面。

## 三项重塑竞争格局的变化

### MCP 标准化工具调用

MCP 全称 Model Context Protocol，由 Anthropic 于 2024 年底提出，用于规范大模型与外部工具、数据源之间的交互。它可以类比为 Agent 工具调用领域的 USB 接口。

MCP 出现之前，不同框架使用各自的工具接口，工具从 LangChain 迁移到 CrewAI 等框架时往往需要重写。本篇统计的十个主流框架中，至少八个已在 2026 年 3 月原生支持 MCP。因此，工具数量与专属工具生态在选型中的权重正在下降。

### A2A 连接不同 Agent

A2A 全称 Agent-to-Agent Protocol，由 Google 于 2025 年提出。MCP 解决 Agent 怎样使用工具，A2A 解决不同 Agent 怎样通信。

本篇把 Microsoft Agent Framework、Google ADK 和 AgentScope 列为原生支持者，把 CrewAI 与 LangGraph 列为通过社区插件提供部分支持的框架。A2A 的价值主要体现在跨框架、跨组织的 Agent 互操作，因此采纳周期可能长于 MCP，但方向是从框架孤岛走向互操作网络。

### 从 Prompt Engineering 转向 Context Engineering

只设计 Prompt 并不足以构成稳定的 Agent 系统。Context Engineering 还需要管理进入上下文窗口的信息质量与结构，并推动框架改进记忆压缩、上下文过滤和动态工具选择。

工具接口逐渐由 MCP 标准化，Agent 通信逐渐由 A2A 标准化后，框架之间更重要的差异转向架构范式与编排能力。

## 十个框架与五种架构范式

本篇总览列出 LangGraph、CrewAI、LlamaIndex、PydanticAI、Agno、Microsoft Agent Framework、OpenAI Agents SDK、Google ADK、Dify 和 AgentScope 十个框架。正文以五种范式分析其中七个框架，另行讨论 Microsoft Agent Framework 与 Google ADK；Agno 出现在总览、趋势与团队建议中，但没有独立展开完整能力分析。

### 图状态机：LangGraph

LangGraph 的取向是“少抽象，多控制”。它用 Node、Edge 和 State 三类基本元素表达复杂执行图，设计灵感来自 Google Pregel 与 Apache Beam。框架不替开发者预设 Agent 的思考、协作、状态模式、编排逻辑和错误处理方式，因此灵活性与学习成本同时较高。

本篇根据实际反馈认为，即使有经验的团队也可能需要两至三周才能写出生产级 LangGraph 代码。其核心能力包括：

- Checkpointer 支持状态持久化与历史状态回溯。
- Interrupt 可以在任意节点暂停，等待人工审批后从断点继续。
- Streaming 支持 Token 级流式传输。

资料列举 Klarna 的客户服务 Agent、Vodafone 的数据工程 AI 助手和 Replit 的代码生成工作流作为应用案例。在来源自己的评分中，LangGraph 的多 Agent 编排、生产部署成熟度、生态、可观测性与定制能力均为 5 分，上手难度为 2 分。这些分数属于来源评价，不是跨项目通用测评。

### 角色驱动：CrewAI

CrewAI 用 Role、Goal 和 Backstory 定义 Agent，强调像组建真实团队一样组织协作。其两种编排方式分别是强调自主协作的 Crews，以及提供事件驱动精确控制路径的 Flows。

本篇记录 CrewAI 当时拥有 4.59 万余 GitHub Stars，超过 10 万名开发者完成官方课程认证，并将其上手难度评为 5 分。易用性的代价是核心框架仍处于 `0.x` 版本、API 变化较频繁，复杂条件分支和状态管理的表达能力弱于 LangGraph。

CrewAI 适合内容创作管线、市场调研和快速原型，不适合需要极致状态控制的金融交易系统或要求精确错误恢复的长时间运行工作流。

### 事件驱动：LlamaIndex 与 AgentScope

资料将 LlamaIndex 的核心理念概括为“数据是 Agent 智能的基石”。它深耕数据连接，目标是把正确数据在正确时间以正确格式送入大模型。资料称其数据层拥有 300 多个连接器，LlamaParse 支持 130 多种文件格式，包括复杂嵌套表格与手写笔记；编排层使用事件驱动、异步优先、步骤化执行的 Workflows 1.0。

本篇把 LlamaIndex 定位于数据密集型 Agent，并列举私募基金处理复杂金融文档、保险公司分析保单、制造业从技术规格书中提取洞察三类场景。

AgentScope 来自阿里巴巴通义实验室，强调透明可控：Prompt、API 调用与决策步骤均对开发者可见。资料列出的三项优势是内置模型微调和 Agent 强化学习微调、同时提供 Python 与 Java 版本，以及原生适配通义千问、阿里云函数计算与钉钉等国内生态。

### SDK 封装：OpenAI Agents SDK 与 PydanticAI

OpenAI Agents SDK 以 Agents、Handoffs 和 Guardrails 三个核心概念保持较低门槛。Handoff 允许 Agent 完成任务后把控制权移交给更合适的 Agent。资料称其可以用五行代码启动，并与 Responses API、微调和蒸馏工具链原生集成，还支持含中断检测与实时流式语音的 Realtime Voice Agents。

本篇同时指出它缺少持久化执行和检查点机制，编排模式有限，不支持并行与循环。这一判断对应资料发布时的版本状态。

PydanticAI 强调类型安全，希望把 FastAPI 的开发体验带给 Agent。输入与输出通过 Pydantic Model 验证，结构化输出解析失败时自动重试；资料称它支持 25 个以上模型提供商，切换模型时无须重写业务逻辑，并内置 TestModel 与 Mock 工具以支持确定性测试。

来源转述的行业评价认为，PydanticAI 可能成为类型安全 Agent 基础设施层的事实标准。这是趋势判断，不是已经确定的结论。

### 低代码平台：Dify

Dify 不是单纯的 Python 库，而是包含拖拽式工作流编辑器、模型管理、知识库管理、API 发布和日志监控的完整 Web 平台。它把复杂 Agent 工作流的构建门槛降低到非技术人员也能参与，但表达能力受可视化编辑器限制，复杂定制最终仍可能回到代码。

本篇列举 Maersk 与 Novartis 作为企业使用案例，并称 Dify 于 2026 年 3 月完成 3000 万美元 Pre-A 轮融资，估值 1.8 亿美元。

同一视频对 Dify 的 GitHub Star 数存在内部差异：旁白称“13.1 万余”，对应画面显示 `120,247`，并标注全球第 51。本篇保留两种记录，不把它们统一为一个数字。

## 两个企业统一型框架

### Microsoft Agent Framework

Microsoft Agent Framework 是 AutoGen 与 Semantic Kernel 的统一继承者。资料将 AutoGen 描述为偏学术前沿的多 Agent 研究框架，当时约有 5.04 万 Stars；Semantic Kernel 偏企业稳定的 AI 编排，当时约有 2.1 万 Stars。合并后的框架把群聊、辩论和反思等多 Agent 模式，与企业安全和遥测能力结合起来。

本篇列出的三项优势是同时支持 .NET 与 Python，原生覆盖 A2A、AG-UI 与 MCP，并与 Azure AI Foundry 深度集成。资料记录其于 2026 年 2 月达到 Release Candidate 状态，并预计在 2026 年第一季度末发布 GA；这属于当时的版本计划。

### Google ADK

Google ADK，即 Agent Development Kit，走多语言路线，覆盖 Python、TypeScript 与 Java，Go 在当时仍处于开发阶段。它用 LlmAgent 处理智能推理，用 SequentialAgent、ParallelAgent 和 LoopAgent 进行确定性编排，并允许通过 BaseAgent 自定义扩展。

Google ADK 与 Vertex AI、Cloud Run 原生集成，因此被本篇视为 Google Cloud 企业的自然选择。资料同时指出，当时框架仍处于 `0.x` 版本，社区规模小于 LangChain，离开 Google 生态后灵活性会下降。

## 四步选型决策树

### 第一步：团队技术栈是什么

- .NET 或 C#：资料直接推荐 Microsoft Agent Framework。
- Java：考虑 Google ADK 或 AgentScope 的 Java 版本。
- TypeScript：考虑 OpenAI Agents SDK 的 JavaScript/TypeScript 版本或 LangGraph.js。
- 多语言混合团队：Google ADK 的语言覆盖面更广。
- 纯 Python 团队：选择最多，需要继续按场景收敛。

### 第二步：核心使用场景是什么

- 数据密集型 RAG 或文档问答：LlamaIndex。
- 角色化多 Agent 协作：CrewAI。
- 复杂、有状态工作流：LangGraph。
- 快速原型或 MVP：CrewAI 或 OpenAI Agents SDK。
- 企业低代码 AI 平台：Dify。
- 国内企业私有化部署：AgentScope 或 Dify。

### 第三步：部署在哪个云上

- Azure：Microsoft Agent Framework。
- Google Cloud：Google ADK。
- 阿里云：AgentScope。
- 自托管或私有化：Dify。
- 没有特定云偏好：回到场景选择。

### 第四步：对大模型是否有偏好

- 深度依赖 OpenAI 生态：OpenAI Agents SDK。
- 依赖 Gemini：Google ADK。
- 使用通义千问或文心一言等国内模型：AgentScope 或 Dify。
- 追求模型无关性：资料强调 PydanticAI 可以在 25 个以上提供商之间切换。

四步完成后，选择通常可以收敛到一两个框架。这套决策树是来源给出的 2026 年 3 月建议，实际采用前仍需按当前版本重新核实。

## “免费”与“生产可用”之间的隐性成本

十个框架的核心代码均开源免费，但生产部署成本并不相同。资料认为：

- LangGraph 的生产部署通常需要付费的 LangGraph Platform。
- LlamaIndex 的高质量文档解析依赖付费的 LlamaParse。
- Dify 的开源版与商业版存在功能差异。
- Microsoft Agent Framework 与 Google ADK 分别深度绑定 Azure 和 Google Cloud 的付费服务。
- PydanticAI 相对轻量，隐性成本较低。

这些结论受框架商业模式和版本变化影响。选型不能只比较许可证，还要在上线前明确托管、可观测性、高质量解析、企业权限与云服务的实际费用。

## 三项演进趋势

### 框架继续整合

微软合并 AutoGen 与 Semantic Kernel 被视为框架数量继续减少的信号。来源判断，独立 AutoGen 已进入维护模式并可能被边缘化；这一判断保留“可能”的不确定性。

### 核心能力模块化

LlamaIndex 把 Workflows 独立成包，说明编排等核心能力可以脱离母框架复用。框架不必继续扩张为包办一切的单体。

### 多模态成为基本能力

OpenAI Agents SDK 的 Realtime Voice、Google ADK 的双向流式音视频与 Agno 的多模态输入，被用来说明语音和视觉正在从附加能力转为 Agent 框架的基本要求。

## 国内生态的差距与优势

本篇认为，AgentScope 与 Dify 在国际社区影响力、英文文档质量和第三方集成广度方面落后于 LangGraph 与 CrewAI；优势则在于 Dify 的社区规模、AgentScope 的模型微调集成、对国产模型的原生支持，以及私有化部署和数据安全合规能力。

这些比较是来源在 2026 年 3 月作出的判断，需要结合组织的监管条件、部署区域和当前产品版本重新评估。

## 面向三类团队的建议

1. **独立开发者与技术探索者**：从 PydanticAI 或 OpenAI Agents SDK 起步，以较平缓的学习曲线建立 Agent 开发认知；需要复杂编排后再转向 LangGraph。
2. **初创公司技术团队**：用 CrewAI 快速验证原型，验证通过后迁移到 LangGraph 构建生产系统；也可以考虑 Agno 的全栈方案，从原型到部署采用同一套框架。
3. **大型企业技术委员会**：先明确云战略。Azure 倾向微软方案，Google Cloud 倾向 Google ADK，多云混合环境可以考虑 LangGraph 与 Dify 组合；同时优先检查 MCP 与 A2A 支持，以保留互操作能力。

## 总结

框架竞争正从工具数量转向架构范式、编排能力、生产成本和生态适配。LangGraph 强在控制，CrewAI 强在易用，LlamaIndex 强在数据，PydanticAI 强在类型安全，Dify 强在低门槛；企业框架和国内框架则分别受云战略、私有化与合规要求影响。

因此，选框架不是选择抽象意义上最好的产品，而是选择最适合团队技术栈、核心场景、部署平台、模型偏好和成本边界的方案。
