---
title: AI Agent：从工具调用到可信行动
created: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - AI Agent
  - 可信执行
  - 综合
---

# AI Agent：从工具调用到可信行动

AI Agent 不是“能调用工具的模型”这么简单。模型负责提出下一步行动，工具接口负责把意图转换为调用，Harness 管理上下文、权限、状态与恢复，Loop 决定何时继续，评估与验证器判断结果是否合格。缺少其中任一层，局部正确都可能在长链中累积成任务失败。（[[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP|AI Agent 基础]]、[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness|Agent Harness]]、[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程]]、[[wiki/syntheses/评估工程：从通用基准到业务质量门|评估工程]]）

## 最小执行链

Function Calling 连接模型与 Agent：模型返回结构化调用请求，Agent 执行本地函数并把结果送回模型。MCP 连接 Agent 与外部服务，可以暴露 Tool、Resource 与 Prompt。两者只解决接口问题，不自动提供权限控制、持久状态、独立验证或停止条件。（[[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP|AI Agent 基础]]）

Pydantic AI 示例进一步表明，工具注册与消息历史也是两件事。`tools` 决定模型能够调用哪些本地函数，`all_messages()` 和 `message_history` 负责跨调用恢复对话；该示例没有实现持久记忆、权限隔离、验证或恢复。（[[wiki/sources/AI Agent 实践：Pydantic AI 工具调用与消息历史|Pydantic AI 实践]]）

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

循环工程资料用每步 95%、连续 20 步约 36% 说明局部成功率的连乘效应；世界模型资料中的 90% 单步合法率、30 步约 4% 展示了同一问题。生产护栏还会发生误报级联：多层过滤串联后，单层看似可接受的误报会压低整体合法流量通过率。（[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程]]、[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|Agent 世界模型]]、[[wiki/sources/评估工程：第七期 从事后评估到生产护栏，差的是挡住还是知道？|生产护栏]]）

因此，增加步骤、Agent、工具或护栏都不是独立收益。每增加一层，都要记录它改变了什么错误率、延迟和恢复能力；没有证据的层只会扩大状态空间和故障面。

## 状态的可信生命周期

可信状态需要经过写入、验证、使用、复核、降权和取代。上下文污染说明错误信息一旦进入窗口会影响后续判断；`STALE` 与 Honest Lying 说明过期或自信错误的记忆会被继续复用；MemRL 则让环境奖励更新记忆的 Q-value，但仍依赖奖励质量。（[[wiki/syntheses/上下文工程：有限窗口中的信息治理|上下文工程]]、[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|Agent 世界模型]]、[[wiki/sources/Agent 记忆：MemRL 运行时强化学习|MemRL]]）

LLM Wiki 的 Supersession 与 Retention／Forgetting 提供文档层治理：新资料不静默覆盖旧资料，而是保留版本、冲突和取代关系。运行时系统也需要同样原则，不能把检索命中或模型自述当作事实确认。（[[wiki/sources/上下文工程：LLM Wiki 的摄取时编译与知识治理|LLM Wiki]]）

## 生成者与放行者必须分离

模型适合生成方案、工具调用和修复建议，确定性检查、测试、类型约束与独立评判器负责放行。HarnessX 的 AEGIS、循环工程的 Evaluator、评估工程的合同驱动架构都指向同一控制原则：灵活生成与严格验收不能由同一条自我评价链代替。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX]]、[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程]]、[[wiki/sources/评估工程：第八期 AI 评估的最后一公里到底长什么样？|合同驱动评估]]）

这套分工同样适用于安全。服务层分类器、拒绝与模型回退只能控制请求入口；Agent 获得文件、命令和生产数据权限后，还需要工具权限、沙箱、审计、人工审批、回滚和 Prompt Injection 防护。（[[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放|Claude 分层开放]]、[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness|Agent Harness]]）

## 采用顺序

1. 先用单次模型调用建立任务基线。
2. 只暴露完成任务所需的最小工具集。
3. 对最大失败来源增加确定性检查。
4. 需要跨步时再引入持久状态、Loop 与恢复。
5. 用 Outcome、Transcript 和环境状态共同评估任务。
6. 只有现有结构无法满足任务时，再增加多 Agent、学习型世界模型或在线经验更新。

框架选型应服从这条链路，而不是反过来。不同框架在工作流、群体协作、数据编排、低代码和生产治理上的生态位不同；选择标准是任务复杂度、团队语言、部署环境和可观测要求。（[[wiki/sources/AI Agent 框架选型：十大框架与五大范式|Agent 框架选型]]）

## 资料链

- [[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP]]
- [[wiki/sources/AI Agent 实践：Pydantic AI 工具调用与消息历史]]
- [[wiki/sources/AI Agent 框架选型：十大框架与五大范式]]
- [[wiki/sources/Agent 记忆：MemRL 运行时强化学习]]
- [[wiki/sources/Agent 世界模型：服务于行动的选择性压缩]]
- [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV]]
- [[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放]]
- [[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- [[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- [[wiki/syntheses/评估工程：从通用基准到业务质量门]]
- [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
