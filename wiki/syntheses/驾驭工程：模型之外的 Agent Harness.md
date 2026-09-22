---
title: 驾驭工程：模型之外的 Agent Harness
updated: 2026-09-22
tags:
  - AI
  - Agent
  - Agent Harness
  - 驾驭工程
  - 综合
---

# 驾驭工程：模型之外的 Agent Harness

Agent Harness 是模型输出与真实执行之间的工程外壳：它组装输入、提供工具、控制授权与执行、保存状态，并组织验证、恢复和审计。判断一个 Harness 是否完整，关键是每项运行责任是否有明确承担者；接通工具、保存消息或增加推理预算，都只能覆盖其中一部分。（[[wiki/sources/驾驭工程：Harness Engineering 运行系统全景|运行系统全景]]、[[wiki/sources/AI Agent：工具调用、MCP 与最小实现|最小 Agent 实现]]）

## 五种来源口径

| 来源 | Harness 的范围 | 这一定义回答什么 |
| --- | --- | --- |
| [[wiki/sources/上下文工程：提示词、上下文与 Harness 的职责边界|Prompt／Context／Harness 对照]] | 以预设工作流、权限和检查约束执行 | 怎样区分任务表达、信息组织和执行治理 |
| [[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程]] | 包住 Prompt、Context 与 Loop 的共同外壳 | 工具、安全、隔离和恢复位于架构何处 |
| [[wiki/sources/驾驭工程：系列完结，下一步该往哪走？|系列收尾篇]] | 模型之外的治理、优化和编排 | 作者怎样总括工程领域；明确不是统一定义 |
| [[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX]] | 可序列化的配置、Processor 与生命周期挂载点 | 怎样把运行外壳变成可组合、可替换的优化对象 |
| [[wiki/sources/驾驭工程：Harness Engineering 运行系统全景|运行系统全景]] | 覆盖任务完整生命周期，Harness 包含 Context、Context 包含 Prompt | 怎样统一任务、工具、状态、验证、恢复和接管 |

这五种口径分别用于职责划分、架构定位、领域总括、具体实现和生命周期管理。引用时保留来源口径，不能把 HarnessX 的实现维度或某位作者的宽泛定义当成所有系统必须采用的标准。

## 从模型提议到验证与恢复

工具链的起点是模型提出动作。Function Calling 规范模型与应用之间的调用结构，MCP 规范应用与外部服务之间的连接；实际操作由客户端或服务执行。Claude Code 的 `tool_use`／`tool_result` 抓包显示，工具结果回到模型后才继续回答，但协议闭环本身不证明环境已经达到目标。（[[wiki/sources/AI Agent：工具调用、MCP 与最小实现|接口边界]]、[[wiki/sources/Claude Code：tool_use、tool_result 与客户端工具闭环|工具闭环抓包]]）

完整运行链还需要以下责任：

| 环节 | Harness 必须处理的问题 | 依据 |
| --- | --- | --- |
| 输入与能力准备 | 当前任务需要哪些规则、证据、工具及参数契约 | [[wiki/sources/上下文工程：提示词、上下文与 Harness 的职责边界|职责边界]] |
| 授权 | 动作是否符合路径、工具、模式与人工审批规则 | [[wiki/sources/Claude Code：权限规则、Permission Mode 与本地放行|权限系统]] |
| 执行与观测 | 在受控环境运行，记录返回、文件变化、通知与轨迹 | [[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解|Runtime 架构]] |
| 独立验证 | 根据测试或环境状态判断结果，而非采用生成者的自我评价 | [[wiki/sources/驾驭工程：Harness Engineering 运行系统全景|生成与验证分离]] |
| 继续、恢复或停止 | 区分传输失败、上下文溢出、输出截断与任务失败，保存可继续的状态 | [[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解|运行闭环与恢复]] |

Anthropic 的 Planner／Generator／Evaluator 与 Aletheia 的 Generator／Verifier／Reviser 支持同一判断：灵活生成与严格放行应由不同角色承担。Harness 执行质量门，[[wiki/syntheses/评估工程：从通用基准到业务质量门|评估工程]]定义合格证据；要跨轮自治，还须由[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程]]接上外部触发、持久状态、预算与停止条件。

故障处理也必须按责任定位。Codex 重连案例只在 HTTPS 可用、WebSocket 链路不稳定时调整传输能力，保留 Responses API 与身份验证；它支持区分传输、API 语义与认证，不能作为所有重连故障的通用修复。（[[wiki/sources/Codex：禁用 WebSocket 解决重复重连|传输故障边界]]）

## 状态必须分层保存

“Agent 有记忆”不足以解释恢复能力。Claude Code Runtime、`/compact` 与 AgentENV 分别展示了不同状态及其保存范围：

| 状态 | 保存目的 | 不能替代什么 |
| --- | --- | --- |
| 当前消息与工作摘要 | 让后续调用理解意图、约束与进度 | 无法单独恢复真实文件和进程 |
| 近期工具消息与附件 | 保持调用／返回配对，重新提供工作材料 | 不是完整执行环境快照 |
| 长期经验与会话检查点 | 跨任务复用知识，或继续已有会话 | 经验记忆不等于当前环境状态 |
| 文件系统、内存、进程和依赖 | 恢复本地执行现场 | 不回滚远程 API、外部事务或已发送消息 |

前两层由[[wiki/sources/Claude Code：compact 上下文压缩与工作现场恢复|压缩与工作现场恢复]]具体说明，长期记忆与会话状态见[[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解|Runtime 架构]]；环境层见[[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV|AgentENV]]。外部副作用仍需幂等键、日志和补偿流程，microVM 快照不会自动撤销已经发生的远程操作。

因此，子上下文隔离、Worktree 与沙箱不能互相替代：它们分别侧重信息、文件和执行环境。信息选择、压缩、缓存及按需加载的具体取舍由[[wiki/syntheses/上下文工程：有限窗口中的信息治理|上下文工程综合]]展开，本页保留“谁保存什么、能恢复到哪里”的运行责任。

## 安全边界不能交给提示词独自承担

Claude Code 权限资料把模型提出动作与客户端放行分开：聊天约束指导模型，规则、Permission Mode 与 Sandbox 控制执行。无人值守也不等于放宽权限；资料中的 `dontAsk` 拒绝未预先允许且需要询问的操作，`bypassPermissions` 则只适合隔离、可重置的环境。具体优先级和模式行为受资料版本边界限制。（[[wiki/sources/Claude Code：权限规则、Permission Mode 与本地放行|权限系统]]）

能力加载与信任判断同样不同。Skill 按需读取只降低初始上下文成本，第三方说明和脚本仍可能接触项目、Shell 与凭据。WebSearch 子 Agent 只拥有搜索工具，缩小了不可信网页直接调用高权限工具的机会，但它返回的误导文字仍可能影响主 Agent；隔离不能替代来源核验。（[[wiki/sources/Claude Code：Skill 渐进式披露与第三方执行边界|Skill 信任边界]]、[[wiki/sources/Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离|搜索隔离]]）

服务门控则控制另一层责任。Fable 5／Mythos 5 资料讨论身份、领域风险、分类器和回退策略；这些措施不能替代工具权限、生产访问控制、审计与回滚。资料中的保守分类器会误伤合法请求，因此门控是安全与可用性的取舍，不能从特定测试得分推出开放环境绝对安全。（[[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放|服务层门控]]）

## 检查器是最小的行动模型

当主要失败来自非法动作时，确定性检查器可以只保留与合法性有关的状态，在执行前拒绝无效动作，无需先建设完整环境模拟器。AutoHarness 的游戏实验支持这种局部做法；它没有证明检查器能判断所有任务是否完成。（[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|检查器与世界模型]]）

检查器仍可能过期或被钻漏洞。应按已观察到的失败选择检查范围，再用真实执行和独立测试校准；动作合法、任务成功与业务合格是不同判定，不能共用一个未经验证的通过信号。

## 从静态外壳到可进化对象

HarnessX 用 Processor 与固定生命周期挂载点分离行为配置，再让 AEGIS 根据轨迹提出修改，由确定性闸门决定是否接受。这把外壳变为可优化对象，也把奖励作弊、遗忘和探索不足带入配置维护。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX]]）

确定性闸门本身仍需检验：资料中的 `pass@2` 可以拒绝明显任务翻转，却看不到成功概率缓慢下降；tau3-Bench 中提醒规则逐次累积，最终形成合规率退化。放行规则可重复执行，不代表其覆盖已经充分。

不同任务对提示、工具或控制策略产生经评测确认的冲突时，多个 Harness 变体与任务路由可以缓解单一配置的反复牺牲；代价是版本、路由、回归测试和审计成本。变体应回应实际冲突，不作为默认架构。

训练协同还有两条不同路线：[[wiki/sources/大模型后训练：RLM Harness 组合泛化|RLM Harness]]通过外移上下文和程序化子调用学习可组合的分解方式；[[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习|SKILLRL]]把轨迹蒸馏为技能并训练模型使用。HarnessX 则共享不同外壳产生的轨迹进行协同训练。三者共同扩大了学习对象，但不意味着不改模型也能获得所有训练收益；训练机制由[[wiki/syntheses/大模型后训练：从模仿到行为选择|后训练综合]]承接。

## 工程收益需要与模型升级比较

运行责任需要保留，实现组件需要复测。Harness 全景资料中，Vercel 删减工具后减少了步骤与 Token，Anthropic 为 Sonnet 4.5 设置的上下文重置在 Opus 4.5 下失去必要性。这支持维护最小充分工具集、删除已失效补偿措施，不支持取消权限、验证或恢复。（[[wiki/sources/驾驭工程：Harness Engineering 运行系统全景|工程收益与模型升级]]）

上述组织案例多来自厂商自述或行业归纳，不能拼接成统一效果证明；BrowseComp 的单／多 Agent 风险差异也只适用于对应配置。Claude Code Runtime 资料缺少源码仓库、Commit 和可复核快照，支持运行职责的结构性判断，不能保证其内部名称和数量适用于其他版本。

可进化外壳至少需要版本化配置、完整轨迹、可验证任务、确定性回退和独立回归集。HarnessX 实验没有独立留出测试集，只覆盖离散文本动作并依赖强根 Agent；RLM 与 SKILLRL 也受底座、任务可分解性或教师模型限制。现有证据支持 Harness 是重要优化对象，不能得出“瓶颈永远不在模型”。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX 限制]]、[[wiki/sources/大模型后训练：RLM Harness 组合泛化|RLM 边界]]、[[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习|SKILLRL 限制]]）

## 实现细节的阅读入口

以下资料保留协议字段、配置条件和当次实验数字；这些细节用于诊断具体系统，不作为 Harness 的固定架构：

- 请求组成与计量：[[wiki/sources/Codex：请求结构、服务端通信与 Token 计量|Codex 请求]]、[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量|Claude Code 请求]]、[[wiki/sources/Claude Code：Thinking 模式、Adaptive 与 Effort|Thinking 控制]]。
- 缓存与配置生命周期：[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本|多轮成本]]、[[wiki/sources/Claude Code：cache_control 断点与 20 Block 前缀回溯|缓存断点]]、[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界|命中边界]]、[[wiki/sources/Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header|第三方缓存兼容]]。
- 能力发现与加载：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持|ToolSearch]]、[[wiki/sources/Claude Code：Skill 渐进式披露与第三方执行边界|Skill]]。
- 模型接入应用的实例：[[wiki/sources/大语言模型：Qwen 3.5 的 MoE、混合注意力与应用演示|Qwen 3.5 与 Cline]]；跨模型的行动闭环见[[wiki/syntheses/AI Agent：从工具调用到可信行动|AI Agent 综合]]。
