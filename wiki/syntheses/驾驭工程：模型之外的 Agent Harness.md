---
title: 驾驭工程：模型之外的 Agent Harness
updated: 2026-09-14
tags:
  - AI
  - Agent
  - Agent Harness
  - 驾驭工程
  - 综合
---

# 驾驭工程：模型之外的 Agent Harness

Agent Harness 是模型输出与真实执行之间的工程外壳。它决定模型看到什么、可以调用什么、怎样跨步骤传递状态、哪些行为被拦截，以及结果如何记录和验证。模型提供推理能力，Harness 把能力约束为可执行、可恢复、可审计的系统行为。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX 专题]]）

## 五种来源口径

本库资料对 Harness 的范围采用五种相关但不完全相同的表述：

1. Prompt、Context 与 Harness 对照资料采用较窄的教学口径：Prompt 解决“怎么问”，Context 解决“怎么记”，Harness 通过预设工作流、权限和质量检查解决“怎么管”。（[[wiki/sources/上下文工程：提示词、上下文与 Harness 的职责边界|三者边界专题]]）
2. 循环工程将 Harness 定义为包住 Prompt、Context 与 Loop 的共同外壳，横切工具、权限、安全、隔离和恢复。（[[wiki/syntheses/循环工程：从逐轮操作到外部调度|循环工程综合]]）
3. 驾驭工程收尾篇采用更宽的作者定义，把模型之外的治理、优化和编排全部归入 Harness Engineering，并明确该术语没有统一定义。（[[wiki/sources/驾驭工程：系列完结，下一步该往哪走？|驾驭工程收尾篇]]）
4. HarnessX 把范围落实为可序列化的一等运行时对象：模型配置与九维行为配置通过 Processor 和固定生命周期挂载点组成可执行 Agent，并可以接受自动进化。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX 专题]]）
5. Harness Engineering 全景资料采用模型外部运行系统口径，并明确给出 Harness 包含 Context、Context 包含 Prompt 的嵌套关系；它把任务、工具、权限、状态、验证、恢复、日志和人类接管统一到运行机制中。（[[wiki/sources/驾驭工程：Harness Engineering 运行系统全景|Harness Engineering 全景]]）

这些口径不是互相覆盖的标准答案。第一种用于区分三个职责，第二种描述架构位置，第三种描述宽泛工程领域，第四种定义一套具体可进化实现，第五种描述贯穿完整任务生命周期的运行系统。

## Harness 的工程职责

综合现有资料，Harness 至少承担六类责任：

- **信息治理**：组装 Prompt、Context、记忆和检索结果，控制进入模型窗口的内容。
- **行动接口**：暴露工具、执行环境和状态修改能力，把模型文本转成实际操作。
- **控制流**：处理步骤、重试、中断、恢复、任务结束和跨轮调度。
- **安全边界**：实施权限、护栏、类型约束和确定性放行规则。
- **观测与评估**：保存轨迹、结果和指标，为诊断、评分及持续改进提供证据。
- **训练桥**：把执行轨迹回流到模型训练，或把 Harness 本身纳入搜索和优化对象。

单次 Harness 执行并不自动构成 Loop；只有外部触发、持久状态、独立验证和停止条件连接起来，系统才具备跨轮自治。Harness 也不能替代业务评测：它可以执行和记录，是否合格仍需独立标准决定。

## 生产实践中的收敛模式

Anthropic 以进度文件、Git 历史和 JSON 功能清单把长任务状态外部化，再将 Planner、Generator 与 Evaluator 分离；Google DeepMind 的 Aletheia 以 Generator、Verifier 与 Reviser 构成相似循环。两者共同支持一项工程判断：生成者的自我评价不能替代独立验证，灵活生成与严格放行应由不同角色承担。（[[wiki/sources/驾驭工程：Harness Engineering 运行系统全景|Harness Engineering 全景]]）

OpenAI 案例进一步把版本化代码库知识、确定性 Linter／结构测试和周期性垃圾回收连接起来；Vercel 移除 80% 工具后反而减少步骤与 Token，说明工具覆盖面不是独立目标，最小充分工具集可以降低 Agent 的选择负担。Stripe Minions 的隔离 Devbox 与 MCP 工具接入则说明，工具能力必须与执行边界同时设计。

这些案例不能合并为固定架构：Anthropic 针对 Sonnet 4.5 的上下文重置机制在 Opus 4.5 下失去必要性，说明补偿性组件必须能够随模型升级而删除。Harness 的长期稳定部分是权限、状态、验证和恢复等系统责任，不是某一代模型对应的具体补丁。

## 模型调用与工具连接

最小 Agent 链路包含两个不同接口。Agent 通过 System Prompt 中的格式约定或 Function Calling 向模型声明工具，模型返回调用请求；Agent 再直接执行本地函数，或作为 MCP Client 调用 MCP Server 暴露的 Tool。工具结果由 Agent 交回模型，模型据此继续判断或生成最终回复。（[[wiki/sources/AI Agent：工具调用、MCP 与最小实现|AI Agent 基础]]）

Function Calling 解决模型与 Agent 之间的结构化调用，MCP 解决 Agent 与外部服务之间的连接。MCP 还可以暴露 Resource 与 Prompt，并不绑定具体模型。这些接口构成 Harness 的行动层，但不会自动提供权限、安全、验证、持久状态、停止或恢复机制。（[[wiki/sources/AI Agent：工具调用、MCP 与最小实现|AI Agent 基础]]）

传输能力同样属于 Harness 的运行配置。Codex 重连资料展示了一项局部调整：`wire_api` 仍为 `responses`，OpenAI 身份验证也保持启用，只把模型提供方的 `supports_websockets` 设为 `false`，便可让请求绕过 WebSocket 并直接进入 HTTPS Streaming。它说明 API 语义、身份验证和传输协议是不同配置维度，故障定位时不能把三者混成“接口不可用”。这项处理仅适用于 HTTPS 可用而 WebSocket 链路不稳定的情况，并受视频所示 Codex 版本约束。（[[wiki/sources/Codex：禁用 WebSocket 解决重复重连|Codex 重连处理]]）

同系列的下一个抓包案例继续展示请求边界：`instructions` 承载基础规则，`developer` 消息注入权限、协作模式、Skill 和插件清单，`user` 消息承载项目规范、环境、历史和当前问题，`tools` 定义行动能力。这些内容分别属于信息治理、安全边界和行动接口，却在一次模型请求中共同消耗上下文。资料对工具数量的 16 与 14 两种标注彼此冲突，因此只支持“工具 schema 是可观测成本”，不支持将某个数量当作 Codex 的固定能力。（[[wiki/sources/Codex：请求结构、服务端通信与 Token 计量|Codex 请求解剖]]）

Claude Code 的抓包呈现出相同职责、不同协议结构：`messages` 的首条用户消息同时携带 Hook、延迟工具名、MCP 指南、Skill 与 `CLAUDE.md`，`system` 保存身份、行为、记忆和环境规则，`tools` 提供已加载能力的完整 Schema。响应则通过 `message_start`、`content_block_delta`、`message_delta` 等 SSE 事件增量返回。本次 `hello` 调用的三个输入计量项合计 30,986 Token，其中 14,145 个为缓存写入、16,835 个为缓存读取；这些数字属于作者个人配置与 `claude-opus-4-7` 单次抓包，不是 Claude Code 的固定上下文成本。（[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量|Claude Code 请求解剖]]）

Thinking 抓包进一步区分模型能力与 Harness 的运行时控制：训练使模型能够生成中间推理，Claude Code 请求中的 `thinking: adaptive` 与 `effort: high` 决定当次调用是否使用该输出通道及大致预算，响应协议再把 Thinking Block 放在 Text Block 前。字段存在不等于任务必然正确；该资料只有一次 TTL 缓存案例，没有 Thinking 开关的同题对照，因此 Harness 仍需用任务结果验证额外推理是否产生实际收益。（[[wiki/sources/Claude Code：Thinking 模式、Adaptive 与 Effort|Claude Code Thinking 模式]]）

客户端工具抓包继续拆开模型与 Harness 的行动边界：Opus 返回 `tool_use` 后以 `stop_reason: tool_use` 等待，Claude Code 才在本地执行 Bash，把输出连同相同 `tool_use_id` 包装为 `tool_result`；模型收到第二次请求后生成 `text` 并以 `end_turn` 结束。模型负责选择动作和解释结果，客户端负责真实执行，但这条协议本身不验证命令是否成功或环境是否达到目标状态。（[[wiki/sources/Claude Code：tool_use、tool_result 与客户端工具闭环|Claude Code 客户端工具闭环]]）

Claude Code 权限资料把“模型提出、客户端执行”的中间门禁进一步展开：本地程序先看工具类型、`allow`／`ask`／`deny` 规则和 Permission Mode，再决定允许、询问、拒绝或交给 `auto` Classifier。确定性规则的优先级为 `deny > ask > allow`；聊天中的约束可能随对话变长或压缩而丢失，不能替代硬规则。`dontAsk` 通过拒绝未预先允许的待询问动作支持无人值守，`bypassPermissions` 则必须放进可重置的隔离环境。这些行为来自作者讲解和画面，没有源码与版本号，只能作为当次资料的实现口径。（[[wiki/sources/Claude Code：权限规则、Permission Mode 与本地放行|Claude Code 权限系统]]）

下一次三轮抓包把 Harness 的稳定结构与多轮成本联系起来：首轮把 48,654 Token 写入缓存，第二轮读取这段前缀并新增写入 24 Token，第三轮读取量随之增长到 48,678 Token。工具定义、System Prompt、Hook、MCP、Skill、`CLAUDE.md` 和历史仍随每轮请求发送，但稳定顺序允许服务端复用前缀计算。该现象只说明当次 Claude Code 请求与 Anthropic 计量字段的关系，不代表其他配置具有相同基数。（[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本|Claude Code 多轮缓存]]）

`cache_control` 抓包把缓存友好的 Harness 结构具体化为三个 Explicit Breakpoint：前两个固定在身份提示和行为准则后，第三个随最新用户消息移动。每个位置都保存累积前缀；新位置 Miss 时最多向前查找 20 个 Block。将稳定内容前置、避免切换模型或中途改变 MCP／Hook，以及用 `defer_loading` 和保持原 Session 前缀的 Compact，都是为了让可复用条目留在回溯范围和相同模型边界内。（[[wiki/sources/Claude Code：cache_control 断点与 20 Block 前缀回溯|Claude Code cache_control]]）

缓存命中不仅由对话内容决定，Harness 的配置生命周期也会改变前缀。资料所示 Claude Code 在启动时读取 MCP、Skill 与 `CLAUDE.md`，通过 `/resume` 或 `/reload-plugins` 重建相应数组后，新配置才进入请求并影响缓存；切换主模型则跨越模型隔离的 KV 缓存。资料建议把其他模型的工作放入独立 Subagent，再把结果交回主对话，这是以额外上下文隔离换取主 Session 前缀稳定的具体策略。（[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界|Claude Code 缓存命中边界]]）

Attribution Header 案例进一步说明，Harness 的客户端证明与第三方缓存可能发生协议错位。Claude Code 的 JavaScript 层先写入 `cch=00000`，Bun 的 Zig 原生层在发送前原位替换真实 `cch`；它可用于证明请求来自真实客户端，却也让第三方代理看到每轮不同的首个 System Block。`CLAUDE_CODE_ATTRIBUTION_HEADER=0` 只适用于已经确认无需官方订阅 Attestation、且确因该字段失去缓存的第三方接入，不能作为所有 Claude Code 环境的默认配置。（[[wiki/sources/Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header|第三方 API 的 cch 缓存失效]]）

ToolSearch 把 Harness 的“能力目录”与“可调用 Schema”进一步分层。客户端先暴露 Deferred Tool 名称，模型选中后，客户端才把完整定义带入下一轮；`defer_loading` 与 `tool_reference` 同时维持工具可见性和缓存前缀。它以一次额外模型往返换取更小初始上下文和更窄选择空间。该链路依赖代理支持 `tool_reference`，因此强制开启 `ENABLE_TOOL_SEARCH=true` 不能自动使任意第三方接口兼容。（[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持|ToolSearch 延迟加载]]）

`/compact` 进一步说明 Harness 的恢复状态不能压成单一摘要。客户端先用禁止工具、固定标签、逐字用户约束和直接引用控制总结，再丢弃 `<analysis>`、保留九节 Summary；下一轮还通过 MessagesToKeep 维护工具消息配对，通过 Attachments 恢复近期文件、Plan 与 Skill。摘要负责远端历史，原始近期消息和材料负责工作现场；二者出现内容重复，是以额外 Token 换取连续性的结果，而非纯文本摘要能够独立完成恢复。（[[wiki/sources/Claude Code：compact 上下文压缩与工作现场恢复|Claude Code /compact]]）

Skill 采用相似的能力目录，却把另一类 Harness 边界暴露得更清楚：客户端只在模型点名后读取 `SKILL.md`，再用 Base directory 让说明书继续定位参考文件和脚本。按需加载控制了上下文成本，却没有把第三方内容变成可信代码；Skill 可以使用同一运行环境中的 Shell、项目文件和凭据，因此来源审计、最小权限和隔离仍必须由 Harness 与用户承担。（[[wiki/sources/Claude Code：Skill 渐进式披露与第三方执行边界|Claude Code Skill 加载]]）

WebSearch 抓包把模型路由、上下文隔离和最小工具权限连成一条实际链路：Opus 4.7 在含 28 个工具的主上下文中决定搜索，Haiku 4.5 在只有 `web_search` 的子上下文中接收 10 条结果和约 30 KB 加密内容，主 Agent 最后只获得 URL、标题与摘要。这样既不让搜索中间材料占据主上下文，也不让不可信网页直接接触 `Bash`、`Edit` 和 `Write`。隔离只能缩小直接攻击面；子 Agent 仍可能返回误导文字，不能代替证据验证。（[[wiki/sources/Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离|Claude Code WebSearch]]）

Pydantic AI 的最小文件管理示例展示了静态 Harness 的最小闭环：`tools` 暴露 `read_file`、`list_files` 与 `rename_file`，`run_sync()` 组织模型和工具调用，应用再保存 `resp.all_messages()` 并通过 `message_history` 重建后续上下文。工具注册没有自动产生跨调用记忆；消息历史也没有提供持久化、权限、验证或恢复。这两部分分别属于行动接口与信息治理，不能合并为一个模糊的“Agent 会记住并执行”。（[[wiki/sources/AI Agent：工具调用、MCP 与最小实现|Pydantic AI 实践]]）

Qwen 3.5 的 Cline 演示提供了另一条最小链路：`qwen3.5-plus` 负责理解需求与生成内容，Cline 通过 OpenAI Compatible 接口连接 DashScope，并在 Act 模式下管理项目文件和执行步骤。模型具备视觉 Agent 与工具规划能力，不等于模型本身承担了 API 密钥、文件权限、执行环境和结果验证；这些仍是外部 Harness 的职责。投篮视频在不同运行中出现“五次”与“七次”的差异，也说明工作流不能把单次模型回答直接当作已验证结果。（[[wiki/sources/大语言模型：Qwen 3.5 的 MoE、混合注意力与应用演示|Qwen 3.5 专题]]）

## 高能力模型需要系统级门控

Claude Fable 5 与 Mythos 5 的资料把同一高能力底座按用户身份、领域风险和访问资格分层开放。Fable 5 在服务层增加分类器、拒绝、自动回退和访问控制，Mythos 5 则只向经过审查的团队开放部分高风险能力。安全边界由此不再只是 System Prompt，而是进入请求分类、模型路由、计费、数据保留和账号授权。（[[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放|Claude Fable 5 与 Mythos 5]]）

这类门控只能处理服务入口。模型接入命令、编辑器、调试器和生产数据后，Harness 仍需控制工具权限、沙箱、审计日志、人工审批、独立验证、回滚与 Prompt Injection。内容过滤回答“能否说”，执行边界回答“能否做”；高能力 Agent 必须同时满足两类约束。

资料中的 Firefox 漏洞利用、自动化越狱、蛋白设计和大代码库迁移数字来自厂商材料与对应评测，不能证明开放环境中的普遍能力或绝对安全。Fable 5 的保守分类器也可能误伤合法安全与生命科学请求，说明分层开放是在能力可用性与双重用途风险之间取舍，不是无损护栏。

## Claude Code 的运行时闭环

Claude Code 架构资料把上述职责落到一个生产 Agent Runtime 中：ReAct 循环在每轮依次准备上下文、流式调用模型、执行工具、收集附件并判断终止；权限检查、Hook、压缩与恢复都处在这条主路径上。异常不是循环外的统一报错，而是分别进入 API 退避、529 处理、输出 Token 恢复、Reactive Compact、上下文排空、模型 Fallback 或无人值守重试。（[[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解|Claude Code Runtime 专题]]）

该案例也说明 Harness 的不同状态不能压成单一“记忆”：当前消息属于短期记忆，任务与投机执行属于工作状态，主题文件属于长期记忆，压缩摘要用于窗口恢复，Checkpoint 则持久化会话。Copy-on-Write Overlay 管理待确认写入，`AsyncLocalStorage` 隔离同进程 Agent，Docker 沙箱处理执行边界；这些机制分别对应信息、进程和文件系统状态。

资料所述实现以 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`、四级 Compact、动态 `tools[]` 和 Tool Search 控制上下文成本，以三种 Agent 执行模型和 Coordinator 组织委托，以 Zod Schema、八源权限、两阶段分类、Hook、文件边界与 Shell 检查构成纵深防御。它没有预建 Embedding 代码索引或 AST 分析，而依赖模型配合 `grep`、`glob` 实时检索，减少索引维护的同时把超大代码库效率留作边界。

这份资料没有给出所分析源码的仓库、Commit、版本号或可复核快照，并且只列出可观测性，没有单独展开其结构。因此，它支持“成熟 Harness 需要完整运行闭环”这一结构性判断，不足以证明视频中的数量和内部名称适用于 Claude Code 的其他版本。

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

## 工程收益需要与模型升级比较

Prompt、Context 与 Harness 对照资料记录了一次作者经验：一套使用多种工程技巧、实际效果不错的 AI 系统，被某个未发布模型在没有这些技巧时直接超过。资料没有公开模型、任务、评测方法和具体差距，因此不能据此认定 Harness 必然失效；它只说明工程投资应持续与更强基础模型的直接结果比较。（[[wiki/sources/上下文工程：提示词、上下文与 Harness 的职责边界|三者边界专题]]）

作者以“小马、马鞍与汽车”表达模型跨代升级可能淘汰局部技巧的判断，并用 The Bitter Lesson 指代这一现象。该资料没有展开概念定义，知识库只保留来源观点，不补写外部解释。即使模型提升减少提示或编排需求，权限、真实执行、审计和独立验证仍属于系统边界，不能仅凭这则未公开案例判定其价值消失。

## 采用边界

可进化 Harness 至少需要版本化配置、完整轨迹、可验证任务、确定性回退和独立回归集。缺少这些条件时，自动修改外壳会把不可观测的手工配置变成不可观测的自动配置，风险反而更大。

HarnessX 的结果没有使用独立留出测试集，只覆盖离散文本动作，并依赖强根 Agent；RLM 和 SKILLRL 也分别受限于特定底座、可分解任务与教师模型。当前资料支持“Harness 是模型之外的重要优化杠杆”，不支持“Agent 的瓶颈永远不在模型”这一无条件结论。（[[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness|HarnessX]]、[[wiki/sources/大模型后训练：RLM Harness 组合泛化|RLM Harness]]、[[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习|SKILLRL]]）

Harness Engineering 全景资料中的组织案例主要由 OpenAI、Anthropic、Google DeepMind、Vercel、Stripe 和 Manus 自述或由行业文章归纳；团队规模、代码量、工具删减、Aletheia 成绩与 BrowseComp 风险数字分别对应不同系统和评测，不能拼接成统一效果证明。尤其是 BrowseComp 中单 Agent 0.24% 与多 Agent 0.87% 的非预期解法发生率，只说明更强编排可能在该设置中放大污染风险，不支持“多 Agent 普遍更危险”的结论。（[[wiki/sources/驾驭工程：Harness Engineering 运行系统全景|Harness Engineering 全景]]）

## 资料链

- [[wiki/sources/上下文工程：提示词、上下文与 Harness 的职责边界]]
- [[wiki/sources/驾驭工程：Harness Engineering 运行系统全景]]
- [[wiki/sources/驾驭工程：系列完结，下一步该往哪走？]]
- [[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness]]
- [[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]]
- [[wiki/sources/Codex：禁用 WebSocket 解决重复重连]]
- [[wiki/sources/Codex：请求结构、服务端通信与 Token 计量]]
- [[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- [[wiki/sources/Claude Code：cache_control 断点与 20 Block 前缀回溯]]
- [[wiki/sources/Claude Code：Thinking 模式、Adaptive 与 Effort]]
- [[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本]]
- [[wiki/sources/Claude Code：tool_use、tool_result 与客户端工具闭环]]
- [[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- [[wiki/sources/Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header]]
- [[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- [[wiki/sources/Claude Code：compact 上下文压缩与工作现场恢复|Claude Code：/compact 上下文压缩与工作现场恢复]]
- [[wiki/sources/Claude Code：权限规则、Permission Mode 与本地放行]]
- [[wiki/sources/Claude Code：Skill 渐进式披露与第三方执行边界]]
- [[wiki/sources/Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离]]
- [[wiki/sources/大模型后训练：RLM Harness 组合泛化]]
- [[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习]]
- [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV]]
- [[wiki/sources/Agent 世界模型：服务于行动的选择性压缩]]
- [[wiki/sources/AI Agent：工具调用、MCP 与最小实现]]
- [[wiki/sources/AI Agent：工具调用、MCP 与最小实现]]
- [[wiki/sources/大语言模型：Qwen 3.5 的 MoE、混合注意力与应用演示]]
- [[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放]]
- [[wiki/syntheses/AI Agent：从工具调用到可信行动]]
- [[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
- [[wiki/syntheses/评估工程：从通用基准到业务质量门]]
