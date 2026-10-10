---
title: AI Agent：八种架构的职责与组合
source: https://www.bilibili.com/video/BV1CrKA6YETu
author: AI架构师Leo
published: 2026-07-18
ingested: 2026-09-23
updated: 2026-10-10
tags:
  - AI
  - AI Agent
  - 架构设计
  - 资料摘要
---

# AI Agent：八种架构的职责与组合

原始资料：[[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构]]。

## 核心结论

作者把八种模式组织成可组合的三层能力：ReAct、Plan-and-Execute、Reflection 负责决策；Tools、Memory、RAG 提供能力；Multi-Agent、Autonomous Loop 组织扩展与持续运行。系统输入目标、上下文和约束，输出文字、行动与状态更新，再把执行结果反馈到下一轮；权限、预算、终止条件和审计构成共同的运行边界。（[[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#三层能力与运行边界|三层能力]]）

这是一套作者用于解释系统职责的分类，不是八个互斥框架，也不是统一行业标准。

## 八种模式及其边界

| 模式 | 主要职责与采用条件 | 需要保留的限制 | 原文定位 |
| --- | --- | --- | --- |
| ReAct | 判断、调用工具、观察结果；适合路线未定的开放任务 | 可能重复调用和持续循环；设置完成、预算与人工停止条件 | [[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#一、ReAct：边做边判断\|ReAct]] |
| Plan-and-Execute | 规划器拆任务图，执行器按依赖逐项完成；适合长任务 | 遇到阻塞要重新规划，完成状态须回传 | [[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#二、Plan-and-Execute：先规划，再执行\|规划与执行]] |
| Multi-Agent | 主管分工，专业角色共享任务状态、产物和证据 | 只在并行与专业视角有价值时采用；增加通信、冲突和重复劳动 | [[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#三、Multi-Agent：通过角色分工协作\|角色协作]] |
| Reflection | 生成、按标准评价、反馈修改；适合有明确质量标准的交付 | 自我评价有共同盲区；建议外部测试、可靠数据与人工审核，并限制轮数 | [[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#四、Reflection：通过反馈改进结果\|反思与反馈]] |
| Tools | 模型选工具和参数，网关校验与执行，结果标准化回传 | 发送、删除、付款等副作用需人工审批、幂等与审计 | [[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#五、Tools：让模型具备受控的操作能力\|受控操作]] |
| Memory | 工作记忆保存当前状态，筛选后写入情景、语义、偏好与规则 | 处理去重、冲突、过期、错误、隐私和删除 | [[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#六、Memory：让状态与经验跨轮延续\|记忆治理]] |
| RAG | 拆问题、改查询、选数据源、检查证据与继续搜索 | 来源冲突需查证；质量受检索、文档与引用可追溯性限制 | [[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#七、RAG：主动寻找可验证的证据\|证据检索]] |
| Autonomous Loop | 保存目标状态，持续观察、决策、行动、评估 | 先具备权限、预算、轮数、检查点、日志、审批和停止机制；错误目标会被放大 | [[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#八、Autonomous Loop：围绕目标持续推进\|自主循环]] |

## 采用顺序与组合

作者建议从 Tools + ReAct 的最小闭环开始，按文档与实时数据、跨会话延续、任务依赖、交付质量和专业分工需求，分别增加 RAG、Memory、Plan-and-Execute、Reflection 和 Multi-Agent；具备治理条件后才开启 Autonomous Loop。每增加一层，都要同步承担评估、成本控制和安全治理。（[[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#从最小闭环开始选择|采用条件]]）

竞品简报示例将规划和主管调度放在外层，各角色内部使用 ReAct，Tools 执行操作，RAG 供给证据，Memory 保存偏好、历史与状态，Reflection 审查并反馈，自主运行时负责定时触发及治理。该示例解释职责组合，来源未提供可复现实现或运行指标，不能据此宣称生产验证通过。（[[raw/sources/应用工程/AI Agent/10 分钟讲透 AI Agent 8 种主流架构#组合示例：持续生成竞品简报|组合示例]]）

## 与既有知识的关系

- [[wiki/syntheses/AI Agent：从工具调用到可信行动]]：补充八种能力的分层与组合条件；与现有工具、状态、验证及边界分工相衔接。
- [[wiki/sources/AI Agent：ReAct 与 Plan-And-Execute 构建模式]]：既有实现说明执行 Agent 内部可用 ReAct，本篇从系统职责进一步说明两者可组合。
- [[wiki/syntheses/循环工程：从逐轮操作到外部调度]]：本篇自主循环示例包含定时触发和目标状态；具体上线验收仍需读取循环工程的独立验证、恢复与停止要求。
- [[wiki/sources/AI Agent 框架选型：十大框架与五大范式]]：八种模式描述能力职责，五种范式描述框架组织方式；分类对象不同，不能按名称一一对应或合并计数。
