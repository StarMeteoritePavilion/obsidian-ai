---
title: AI Agent：从工具调用到可信行动
created: 2026-09-04
updated: 2026-09-23
tags:
  - AI
  - AI Agent
  - 可信执行
  - 综合
---

# AI Agent：从工具调用到可信行动

AI Agent 不是“能调用工具的模型”这么简单。模型负责提出下一步行动，工具接口负责把意图转换为调用，Harness 管理上下文、权限、状态与恢复，Loop 决定何时继续，评估与验证器判断结果是否合格。缺少其中任一层，局部正确都可能在长链中累积成任务失败。（[[wiki/sources/AI Agent：工具调用、MCP 与最小实现|AI Agent 基础]]、[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness|Agent Harness]]、[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程]]、[[wiki/syntheses/评估工程：从通用基准到业务质量门|评估工程]]）

## 最小执行链

Function Calling 连接模型与 Agent：模型返回结构化调用请求，Agent 执行本地函数并把结果送回模型。MCP 连接 Agent 与外部服务，可以暴露 Tool、Resource 与 Prompt。两者只解决接口问题，不自动提供权限控制、持久状态、独立验证或停止条件。（[[wiki/sources/AI Agent：工具调用、MCP 与最小实现|AI Agent 基础]]）

Claude Code 的 Bash 抓包把这条抽象链路展开为两个模型请求：Opus 先用 `tool_use` 给出工具名、唯一 ID 和参数，客户端执行 `git status` 与 `git diff`，再以相同 `tool_use_id` 把文本输出包装成 `tool_result`；模型读取结果后返回 `text` 并以 `end_turn` 结束。ID 只负责调用与结果配对，不能代替对命令退出状态和文件结果的验证。（[[wiki/sources/Claude Code：tool_use、tool_result 与客户端工具闭环|Claude Code 客户端工具闭环]]）

Pydantic AI 示例进一步表明，工具注册与消息历史也是两件事。`tools` 决定模型能够调用哪些本地函数，`all_messages()` 和 `message_history` 负责跨调用恢复对话；该示例没有实现持久记忆、权限隔离、验证或恢复。（[[wiki/sources/AI Agent：工具调用、MCP 与最小实现|Pydantic AI 实践]]）

## ReAct 与 Plan-And-Execute 组织不同层级的循环

ReAct 把一次循环组织为 Thought、Action、Observation，直到模型返回 Final Answer。模型只提出工具请求，Agent 主程序解析请求、执行函数并把结果加入消息历史。系统提示词可以约定这套输出协议，却不能代替执行权限和结果验证。（[[wiki/sources/AI Agent：ReAct 与 Plan-And-Execute 构建模式|ReAct 最小实现]]）

Plan-And-Execute 在执行循环外增加显式计划：Plan 模型产生初始步骤，执行 Agent 完成当前步骤，Re-Plan 模型依据执行记录返回新计划或最终答案。执行 Agent 内部仍可使用 ReAct，因此两者不是必须二选一的同层方案。显式规划增加了可见状态，也同时增加了计划、执行记录和终止判断需要保持一致的责任。（[[wiki/sources/AI Agent：ReAct 与 Plan-And-Execute 构建模式|Plan-And-Execute]]、[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程]]）

该 ReAct 示例还说明，任务目录和执行权限需要分别核验：目录参数只参与提示词构造，文件工具未校验路径范围，命令工具未切换至该目录。终端执行前的人工确认也不覆盖文件写入，因此不能把指定项目目录视为沙箱。（[[wiki/sources/AI Agent：ReAct 与 Plan-And-Execute 构建模式#目标目录与权限边界|目录参数的实际作用]]）

复现实例时，先固定代码与依赖条件，再判断执行是否成功。本例的作者 ReAct 项目与官方 Plan-And-Execute Notebook 使用不同凭据和运行环境；原教程网址已跳转，具体版本与准备步骤见 [[wiki/sources/AI Agent：ReAct 与 Plan-And-Execute 构建模式#实践入口与复现状态|实践入口与复现状态]]。

## 单任务执行循环与长期 Loop 的边界

以下是依据本库两组资料作出的综合区分：2025 年 ReAct／Plan-And-Execute 资料讲解一次任务内部怎样继续执行；2026 年循环工程资料讨论一次执行结束后怎样触发、验证并接续下一轮。“长期 Loop”沿用本库循环工程资料的工作定义，不是对所有框架的统一分类，也不意味着 ReAct 或 Plan-And-Execute 不能扩展为长期系统。

| 比较项 | 本文 ReAct／Plan-And-Execute 示例 | 本库长期 Loop 的要求 |
| --- | --- | --- |
| 继续的触发 | 当前任务内，收到 Observation 后继续调用模型，或执行后进入 Re-Plan | 根据时间、事件或持久状态触发下一轮；不能只依赖人的下一条消息 |
| 状态的用途 | 消息历史支持下一次模型调用；计划与执行记录支持当前任务的重规划 | 在对话之外保存任务、尝试、已验证结果与未完成项，下一轮读取后接续 |
| 完成的依据 | 模型返回 Final Answer，或 Re-Plan 返回最终答案，结束当前流程 | 独立 Checker 根据实际结果与验收标准决定是否通过，再决定继续、返修或停止 |
| 中断与资源边界 | 有内部循环不代表已有跨运行恢复、回退和整体预算治理 | 明确轮数、时间、Token 与重试预算，并验证停止、可信状态恢复和人工接管 |

左列依据 [[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 任务路径修订版）#三个工具与一个主循环|ReAct 主循环]]及其 [[raw/sources/应用工程/AI Agent/Agent 的概念、原理与构建模式 —— 从零打造一个简化版的 Claude Code（2026-09-22 任务路径修订版）#Plan-And-Execute：先规划，再动态调整|Plan-And-Execute 流程]]；右列依据 [[raw/sources/应用工程/循环工程/Loop 的组件、搭建方法与上线检查#五步搭建一个有边界的循环|搭建方法]]、[[raw/sources/应用工程/循环工程/Loop 的组件、搭建方法与上线检查#Maker与Checker分离|独立验证]]及 [[raw/sources/应用工程/循环工程/Loop 的组件、搭建方法与上线检查#六项上线检查|上线检查]]。

Re-Plan 的职责是利用执行记录更新计划或结束回答。即使单独调用另一个模型，也不能仅凭这个角色名称认定它已具备独立 Checker 的证据与职责隔离。同样，保存了计划或消息对象，只能说明当前执行有状态，不能据此推断进程重启后能够恢复；需要单独检查持久化与恢复实现。（[[wiki/sources/AI Agent：ReAct 与 Plan-And-Execute 构建模式|执行与重规划]]、[[wiki/sources/循环工程：组件、搭建与上线检查|验证与持久化]]）

衔接两者时，可以把一个有终点的 ReAct 或 Plan-And-Execute 任务作为长期 Loop 的执行环节：先验证一次执行，再接入发现与触发、独立验收、状态写回，以及失败回退和预算停止。外层读取已验证状态决定下一轮，内层负责把当前工作项推进到明确结果；是否需要长期 Loop 取决于跨轮接续需求，不能只因任务步骤多就直接增加调度层。（综合判断，依据 [[raw/sources/应用工程/循环工程/Loop 的组件、搭建方法与上线检查#五步搭建一个有边界的循环|五步搭建]]；完整要求见 [[wiki/syntheses/循环工程：从逐轮操作到外部调度#循环成立的条件|循环成立的条件]]。）

## 四类状态不能统称为记忆

| 状态 | 保存内容 | 主要风险 | 资料入口 |
| --- | --- | --- | --- |
| 请求与推理状态 | 消息历史、工具结果、KV Cache | 窗口膨胀、错误上下文延续 | [[wiki/syntheses/上下文工程：有限窗口中的信息治理|上下文工程]] |
| 经验状态 | 情景记忆、技能、历史效用 | 过期、污染、错误反馈强化 | [[wiki/sources/Agent 记忆：MemRL 运行时强化学习|MemRL]] |
| 判断状态 | 当前对象、因果关系与动作预测 | 初始因果图错误，反复搜索同一失败路径 | [[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|Agent 世界模型]] |
| 环境状态 | 文件、进程、依赖、数据库与操作系统状态 | 消息恢复后外部世界无法复原 | [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV|AgentENV]] |

上下文中存在正确资料，不表示当前判断已经采用它；记忆曾经正确，也不表示现在仍然有效。世界模型和经验库都需要记录来源、验证状态、时间与失效条件。AgentENV 则解决另一层问题：它保存 Agent 正在操作的外部世界，不保存模型参数或内部思考。

## 最小世界模型从失败来源出发

世界模型可以是学习型预测器，也可以只是一段动作合法性检查代码。资料所述公开下棋实验中，单步合法率 90% 连续 30 步后，整局全部合法的概率约为 4%；AutoHarness 生成检查器后，在对应实验中把合法动作率推近 100%。（[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|Agent 世界模型]]）

这并不意味着检查器等于环境真理。用于训练或优化后，Agent 可能利用判定漏洞；规则变化也会让检查器过期。正确顺序是先找出最大的失败来源，再构造能够拦住它的最小检查范围，并让真实执行和独立评测复核结果。

## 长链可靠性取决于组合而非单点

Harness 全景资料用每步 95%、连续 20 步约 36% 说明局部成功率的连乘效应。这是同等条件下单步成功率连续相乘的简化假设示例（`0.95^20 ≈ 36%`），不是实测系统可靠性。（[[wiki/sources/驾驭工程：Harness Engineering 运行系统全景|Harness 全景]]）世界模型资料中的 90% 单步合法率、30 步约 4% 展示了同一问题。生产护栏还会发生误报级联：多层过滤串联后，单层看似可接受的误报会压低整体合法流量通过率。（[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|Agent 世界模型]]、[[wiki/sources/评估工程：第七期 从事后评估到生产护栏，差的是挡住还是知道？|生产护栏]]）

因此，增加步骤、Agent、工具或护栏都不是独立收益。每增加一层，都要记录它改变了什么错误率、延迟和恢复能力；没有证据的层只会扩大状态空间和故障面。

## 状态的可信生命周期

可信状态需要经过写入、验证、使用、复核、降权和取代。上下文污染说明错误信息一旦进入窗口会影响后续判断；`STALE` 与 Honest Lying 说明过期或自信错误的记忆会被继续复用；MemRL 则让环境奖励更新记忆的 Q-value，但仍依赖奖励质量。（[[wiki/syntheses/上下文工程：有限窗口中的信息治理|上下文工程]]、[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|Agent 世界模型]]、[[wiki/sources/Agent 记忆：MemRL 运行时强化学习|MemRL]]）

LLM Wiki 的 Supersession 与 Retention／Forgetting 提供文档层治理：新资料不静默覆盖旧资料，而是保留版本、冲突和取代关系。运行时系统也需要同样原则，不能把检索命中或模型自述当作事实确认。（[[wiki/sources/上下文工程：LLM Wiki 的摄取时编译与知识治理|LLM Wiki]]）

## 生成者与放行者必须分离

模型适合生成方案、工具调用和修复建议，确定性检查、测试、类型约束与独立评判器负责放行。HarnessX 的 AEGIS、循环工程的 Evaluator、评估工程的合同驱动架构都指向同一控制原则：灵活生成与严格验收不能由同一条自我评价链代替。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX]]、[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程]]、[[wiki/sources/评估工程：第八期 AI 评估的最后一公里到底长什么样？|合同驱动评估]]）

这套分工同样适用于安全。服务层分类器、拒绝与模型回退只能控制请求入口；Agent 获得文件、命令和生产数据权限后，还需要工具权限、沙箱、审计、人工审批、回滚和 Prompt Injection 防护。（[[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放|Claude 分层开放]]、[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness|Agent Harness]]）

Claude Code 的本地权限案例给出了一条具体放行链：模型返回工具调用后，客户端依据工具类型、确定性规则和 Permission Mode 决定是否执行。`auto` Classifier 可以参考对话减少机械审批，但对话约束可能随上下文压缩而丢失；长期硬边界需要 `deny`，跳过大多数提示的 `bypassPermissions` 需要可重置沙盒。由此可见，模型行为指导、客户端权限与执行环境隔离是三层不同控制，不能互相替代。（[[wiki/sources/Claude Code：权限规则、Permission Mode 与本地放行|Claude Code 权限系统]]）

## 反复犯错时怎样定位责任层

先保存一次失败的实际输入、工具参数与返回、环境变化、最终结果和验收记录，再定位最早出现偏差的环节。下表把本页的状态分层转成排查入口，不根据“反复犯错”直接认定需要更强模型、更多记忆或更多 Agent。（[[wiki/sources/评估工程：第六期 Agent 评估为什么比 LLM 评估难一个数量级？|轨迹与结果证据]]）

| 失败证据 | 优先检查哪一层 | 核验与处理入口 |
| --- | --- | --- |
| 约束或上一轮错误返回未进入当前请求 | 上下文选择与压缩 | 对比压缩前后内容，保留锚点和未解决失败证据；[[wiki/syntheses/上下文工程：有限窗口中的信息治理#压缩：摘要历史，恢复工作现场|历史与现场恢复]] |
| 检索到过去有效的规则，当前环境却已变化 | 经验的时效与验证状态 | 回查来源、适用条件和失效时间，修正过期记忆；[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩#三层结构与记忆时效|记忆时效]] |
| 正确资料已经可见，仍沿错误因果关系行动 | 当前判断与动作预测 | 用真实返回检验“这一步会改变什么”，核对初始对象和流程；[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|世界模型与最小检查器]] |
| 对话恢复了，文件、进程或外部事务没有恢复 | 执行环境与副作用 | 对账实际状态；消息摘要不能代替环境检查点，快照不能撤回远程事务；[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness#状态必须分层保存|状态恢复边界]] |
| Agent 自称成功，测试或后端结果仍失败 | 验收与停止判断 | 检查独立标准、工具退出状态和实际 Outcome；[[wiki/syntheses/评估工程：从通用基准到业务质量门#Agent 评估：从文本扩展到状态|结果评估]] |
| 条件与证据已核实，任务仍稳定失败 | 能力与反馈的适配 | 回到单次任务基线，判断示范、检索或训练能解决什么；[[wiki/syntheses/大模型后训练：从模仿到行为选择#先判断问题属于哪一层|训练采用条件]] |

修复后重放原失败任务，并补入相近边界用例，分别检查结果正确、行动合规和状态可恢复。错误不再出现在一条演示中，只说明该用例通过；持续可靠性仍由独立评估与回归记录支持。（[[wiki/syntheses/评估工程：从通用基准到业务质量门#Agent 评估：从文本扩展到状态|回归与多次运行]]、[[wiki/syntheses/循环工程：从逐轮操作到外部调度#从能运行到能上线|停止与恢复检查]]）

## 八种模式怎样组合

2026 年 7 月的架构专题把八种模式组织为决策、能力与运行三层：ReAct、Plan-and-Execute、Reflection 决定行动与改进；Tools、Memory、RAG 提供操作、状态与证据；Multi-Agent、Autonomous Loop 负责角色协作和持续推进。权限、预算、终止条件与审计贯穿三层。这是该来源的职责分类，八种模式可以组合。（[[wiki/sources/AI Agent：八种架构的职责与组合#核心结论|八种模式的职责]]）

采用条件应逐项判断：访问外部系统时先建立 Tools + ReAct；依赖文档或实时数据再增加 RAG；需要跨会话延续再增加 Memory；步骤多且依赖明确再增加 Plan-and-Execute；有明确质量标准再加入 Reflection；确需专业分工才增加 Multi-Agent；治理和停止机制齐备后再启用 Autonomous Loop。增加 Agent 数量会增加通信、上下文冲突与重复劳动，模型反思仍需外部证据约束。（[[wiki/sources/AI Agent：八种架构的职责与组合#八种模式及其边界|职责与边界]]、[[wiki/sources/AI Agent：八种架构的职责与组合#采用顺序与组合|采用条件]]）

结合既有框架选型资料，能力模式与框架范式需要分开：本篇八种模式回答系统需要承担哪些职责，框架资料中的图状态机、角色驱动、事件驱动、SDK 封装和低代码则描述组织与交付方式。它们不构成一一映射，不能将两份来源的分类数量拼成统一架构总数。（综合判断，依据 [[wiki/sources/AI Agent：八种架构的职责与组合|八种模式]]、[[wiki/sources/AI Agent 框架选型：十大框架与五大范式#五种范式与生态位|五种框架范式]]。）

## 采用顺序

1. 先用单次模型调用建立任务基线。
2. 只暴露完成任务所需的最小工具集。
3. 对最大失败来源增加确定性检查。
4. 单任务内按需采用 ReAct 或 Plan-And-Execute；需要跨轮或跨运行接续时，再补齐持久状态、外部触发、独立验收与恢复。
5. 用 Outcome、Transcript 和环境状态共同评估任务。
6. 只有现有结构无法满足任务时，再增加多 Agent、学习型世界模型或在线经验更新。

框架选型应服从这条链路，而不是反过来。不同框架在工作流、群体协作、数据编排、低代码和生产治理上的生态位不同；选择标准是任务复杂度、团队语言、部署环境和可观测要求。（[[wiki/sources/AI Agent 框架选型：十大框架与五大范式|Agent 框架选型]]）

## 资料链

- [[wiki/sources/AI Agent：八种架构的职责与组合]]

- [[wiki/sources/AI Agent：工具调用、MCP 与最小实现]]
- [[wiki/sources/AI Agent：ReAct 与 Plan-And-Execute 构建模式]]
- [[wiki/sources/AI Agent 框架选型：十大框架与五大范式]]
- [[wiki/sources/Claude Code：tool_use、tool_result 与客户端工具闭环]]
- [[wiki/sources/Claude Code：权限规则、Permission Mode 与本地放行]]
- [[wiki/sources/Agent 记忆：MemRL 运行时强化学习]]
- [[wiki/sources/Agent 世界模型：服务于行动的选择性压缩]]
- [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV]]
- [[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放]]
- [[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- [[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- [[wiki/syntheses/评估工程：从通用基准到业务质量门]]
- [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
