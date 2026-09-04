---
title: AI Agent 框架选型：十大框架与五大范式
source: https://www.bilibili.com/video/BV1SqAVzLEci
author: 唐国梁Tommy
published: 2026-03-19
ingested: 2026-09-02
updated: 2026-09-03
tags:
  - AI
  - AI Agent
  - 框架选型
  - 资料摘要
---

# AI Agent 框架选型：十大框架与五大范式

原始资料：[[raw/sources/应用工程/AI Agent/2026 AI Agent框架终极指南：从入门到生产部署的选型地图，10大框架五大范式，一期全讲透|2026 AI Agent 框架终极指南]]

## 核心结论

Agent 框架没有脱离场景的统一最优解。选型应依次检查团队技术栈、核心使用场景、部署云平台和模型偏好，再把许可证之外的托管、解析、可观测性、权限与云服务成本纳入判断。

资料认为，MCP、A2A 与 Context Engineering 正在削弱专属工具数量的差异，把竞争重点推向架构范式与编排能力。该判断以及所有版本、Star 数、融资与生态结论均对应 2026 年 3 月，不能直接作为当前状态。

## 五种范式与生态位

- **图状态机**：LangGraph 以 Node、Edge、State 提供高控制力，适合复杂有状态工作流，但学习成本高。
- **角色驱动**：CrewAI 以 Role、Goal、Backstory 组织协作，适合内容管线、调研与原型，但复杂状态控制有限。
- **事件驱动**：LlamaIndex 强在数据连接与文档处理；AgentScope 强调过程透明、微调集成和国内生态。
- **SDK 封装**：OpenAI Agents SDK 以 Agents、Handoffs、Guardrails 保持简洁；PydanticAI 以类型验证、自动重试和 TestModel 支持确定性测试。
- **低代码平台**：Dify 提供可视化工作流、模型、知识库、API 与日志管理，以低门槛换取定制上限。

Microsoft Agent Framework 与 Google ADK 作为大厂企业统一型框架另行讨论：前者继承 AutoGen 与 Semantic Kernel 并深度集成 Azure，后者提供多语言 Agent 开发并深度集成 Google Cloud。

## 四步决策树

1. **技术栈**：.NET/C# 看 Microsoft Agent Framework，Java 看 Google ADK 或 AgentScope，TypeScript 看 OpenAI Agents SDK 或 LangGraph.js，多语言团队看 Google ADK，Python 再按场景选择。
2. **场景**：数据密集型 RAG 看 LlamaIndex，角色协作看 CrewAI，复杂有状态流程看 LangGraph，快速原型看 CrewAI 或 OpenAI Agents SDK，低代码看 Dify，国内私有化看 AgentScope 或 Dify。
3. **部署平台**：Azure 对应微软方案，Google Cloud 对应 Google ADK，阿里云对应 AgentScope，自托管与私有化对应 Dify。
4. **模型偏好**：OpenAI 对应 OpenAI Agents SDK，Gemini 对应 Google ADK，国内模型对应 AgentScope 或 Dify，模型无关性对应 PydanticAI。

这套映射属于来源建议，不是排他性规则；技术、场景、云平台与模型约束冲突时，需要结合优先级重新权衡。

## 生产成本与演进方向

核心代码开源免费不等于生产环境无成本。资料提醒关注 LangGraph Platform、LlamaParse、Dify 商业功能以及 Azure、Google Cloud 等付费依赖，并把 PydanticAI 视为相对轻量的选择。

来源提出三项趋势：框架数量继续整合，核心能力从母框架中模块化拆分，语音和视觉成为 Agent 的基本能力。国内框架在国际社区、英文文档和第三方集成方面存在差距，但在国产模型、私有化和合规方面具有适配优势。

## 来源内部边界

- 总览列出 LangGraph、CrewAI、LlamaIndex、PydanticAI、Agno、Microsoft Agent Framework、OpenAI Agents SDK、Google ADK、Dify、AgentScope 十个框架，但 Agno 只出现在总览、趋势与团队建议中，没有独立能力分析。
- Dify Star 数在同一视频中不一致：旁白称 13.1 万余，画面显示 `120,247`；本库不统一这两个数字。
- LangGraph、CrewAI 等评分与“事实标准”判断均来自作者评价，不是统一基准测试结果。
- 框架版本、协议支持、云绑定和商业功能变化较快，实际选型前必须重新核对官方文档。

## 关联

- Context Engineering：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Loop 与编排：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- Agent 评估：[[wiki/sources/评估工程：第六期 Agent 评估为什么比 LLM 评估难一个数量级？]]
