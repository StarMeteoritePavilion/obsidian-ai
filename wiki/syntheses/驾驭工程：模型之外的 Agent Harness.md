---
title: 驾驭工程：模型之外的 Agent Harness
updated: 2026-09-04
tags:
  - AI
  - Agent
  - Agent Harness
  - 驾驭工程
  - 综合
---

# 驾驭工程：模型之外的 Agent Harness

Agent Harness 是模型输出与真实执行之间的工程外壳。它决定模型看到什么、可以调用什么、怎样跨步骤传递状态、哪些行为被拦截，以及结果如何记录和验证。模型提供推理能力，Harness 把能力约束为可执行、可恢复、可审计的系统行为。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX 专题]]）

## 三种来源口径

本库资料对 Harness 的范围采用三种相关但不完全相同的表述：

1. 循环工程将 Harness 定义为包住 Prompt、Context 与 Loop 的共同外壳，横切工具、权限、安全、隔离和恢复。（[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程综合]]）
2. 驾驭工程收尾篇采用更宽的作者定义，把模型之外的治理、优化和编排全部归入 Harness Engineering，并明确该术语没有统一定义。（[[wiki/sources/驾驭工程：系列完结，下一步该往哪走？|驾驭工程收尾篇]]）
3. HarnessX 把范围落实为可序列化的一等运行时对象：模型配置与九维行为配置通过 Processor 和固定生命周期挂载点组成可执行 Agent，并可以接受自动进化。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX 专题]]）

这些口径不是互相覆盖的标准答案。第一种描述架构位置，第二种描述宽泛工程领域，第三种定义一套具体可进化实现。

## Harness 的工程职责

综合现有资料，Harness 至少承担六类责任：

- **信息治理**：组装 Prompt、Context、记忆和检索结果，控制进入模型窗口的内容。
- **行动接口**：暴露工具、执行环境和状态修改能力，把模型文本转成实际操作。
- **控制流**：处理步骤、重试、中断、恢复、任务结束和跨轮调度。
- **安全边界**：实施权限、护栏、类型约束和确定性放行规则。
- **观测与评估**：保存轨迹、结果和指标，为诊断、评分及持续改进提供证据。
- **训练桥**：把执行轨迹回流到模型训练，或把 Harness 本身纳入搜索和优化对象。

单次 Harness 执行并不自动构成 Loop；只有外部触发、持久状态、独立验证和停止条件连接起来，系统才具备跨轮自治。Harness 也不能替代业务评测：它可以执行和记录，是否合格仍需独立标准决定。

## 模型调用与工具连接

最小 Agent 链路包含两个不同接口。Agent 通过 System Prompt 中的格式约定或 Function Calling 向模型声明工具，模型返回调用请求；Agent 再直接执行本地函数，或作为 MCP Client 调用 MCP Server 暴露的 Tool。工具结果由 Agent 交回模型，模型据此继续判断或生成最终回复。（[[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP|AI Agent 基础]]）

Function Calling 解决模型与 Agent 之间的结构化调用，MCP 解决 Agent 与外部服务之间的连接。MCP 还可以暴露 Resource 与 Prompt，并不绑定具体模型。这些接口构成 Harness 的行动层，但不会自动提供权限、安全、验证、持久状态、停止或恢复机制。（[[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP|AI Agent 基础]]）

## 检查器是最小的行动模型

当 Agent 的主要失败来自非法动作时，Harness 不一定需要构造完整环境模拟器，一个确定性检查器就可以承担最小世界模型：它保留与行动合法性有关的状态，并在执行前拒绝无效动作。资料所述 AutoHarness 让 Agent 自动合成代码检查器，在对应游戏实验中把合法动作率推近 100%；易犯规游戏的对比中，大模型无检查器为 71%，小模型配检查器为 94%。（[[wiki/sources/Agent 世界模型：服务于行动的选择性压缩|Agent 世界模型]]）

检查器只是环境的选择性压缩，不是环境真理。用于训练或自动优化后，Agent 可能钻其判定漏洞；规则变化也会让检查器过期。因此，Harness 应先按最大失败来源选择最小检查范围，再用真实执行、独立测试和版本治理约束它。

## 执行环境也需要独立生命周期

Harness 的“隔离与恢复”不能只停留在抽象职责。长时程 Agent 会修改文件、启动进程、安装依赖和改变数据库，消息历史或工具返回无法单独重建这些状态。Kimi K3 的 AgentENV 以 Firecracker microVM 隔离操作系统内核，并提供 Pause and Resume、Fork 与 Snapshot，使执行环境可以暂停释放资源、为奖励评判复制无副作用分支，并从定期恢复点继续。（[[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV|Kimi K3 AgentENV]]）

这与 Worktree 或子上下文隔离的粒度不同：前者主要隔离文件与信息，microVM 沙箱还覆盖内存、进程、内核和虚拟硬件。AgentENV 也只负责外部环境状态；模型版本、KV Cache、请求状态和策略更新仍由其他训练基础设施管理。一个完整 Harness 需要明确这些状态分别由谁保存，不能用单一“记忆”概念代替。

## 从静态外壳到可进化对象

静态 Harness 的问题不只是不够灵活，还包括组件纠缠和运行轨迹浪费。HarnessX 以窄接口 Processor、八个生命周期挂载点和九维行为配置建立组合边界，使 Prompt、工具、记忆、控制、安全、可观测性和训练桥可以独立修改。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX 专题]]）

AEGIS 再把 Harness 配置视为状态、修改视为动作、轨迹与验证器视为反馈，并由确定性闸门决定是否接受。这个“运行对偶”揭示 Harness 自动优化与强化学习共享奖励作弊、灾难性遗忘和探索不足等风险。

相应的治理原则是：模型负责提出假设和修改，类型、测试、验证器与确定性规则负责放行。生成者的灵活性不能代替执行边界的确定性。（[[wiki/syntheses/评估工程：从通用基准到业务质量门|评估工程综合]]）

## 单一配置与变体隔离

全局 Harness 适合任务需求一致的场景；当不同任务需要互相冲突的 Prompt、工具或控制策略时，继续修改单一配置会形成跷跷板。HarnessX 的变体隔离允许保留多个配置，再由路由器把任务交给更适合的变体。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX 专题]]）

多变体提高适配能力，也增加版本、路由、回归测试和审计成本。只有在冲突已经由轨迹和评测证实后才值得引入，不能把配置分岔当作默认架构。

## 与模型训练协同

RLM Harness 证明，Context Offloading 和 Programmatic Subcalls 可以让长任务通过局部分布内调用实现组合泛化；SKILLRL 把成功与失败轨迹蒸馏为外部技能，再用 SFT 和 RL 教模型使用；HarnessX 则让多个外壳版本成为结构化探索器，并用 Cross-Harness GRPO 从共享轨迹训练模型。（[[wiki/sources/大模型后训练：RLM Harness 组合泛化|RLM Harness]]、[[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习|SKILLRL]]、[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX]]）

三条路线共同说明，Agent 的学习对象可以从孤立模型扩展到“模型＋外部执行结构”。但 Harness 只能补充信息组织、工具和行为约束，无法无限弥补模型缺失的推理能力；反过来，模型变强也不能自动修复坏工具、错误权限和不可靠验证器。

## 采用边界

可进化 Harness 至少需要版本化配置、完整轨迹、可验证任务、确定性回退和独立回归集。缺少这些条件时，自动修改外壳会把不可观测的手工配置变成不可观测的自动配置，风险反而更大。

HarnessX 的结果没有使用独立留出测试集，只覆盖离散文本动作，并依赖强根 Agent；RLM 和 SKILLRL 也分别受限于特定底座、可分解任务与教师模型。当前资料支持“Harness 是模型之外的重要优化杠杆”，不支持“Agent 的瓶颈永远不在模型”这一无条件结论。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX]]、[[wiki/sources/大模型后训练：RLM Harness 组合泛化|RLM Harness]]、[[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习|SKILLRL]]）

## 资料链

- [[wiki/sources/驾驭工程：系列完结，下一步该往哪走？]]
- [[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness]]
- [[wiki/sources/大模型后训练：RLM Harness 组合泛化]]
- [[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习]]
- [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV]]
- [[wiki/sources/Agent 世界模型：服务于行动的选择性压缩]]
- [[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP]]
- [[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- [[wiki/syntheses/评估工程：从通用基准到业务质量门]]
