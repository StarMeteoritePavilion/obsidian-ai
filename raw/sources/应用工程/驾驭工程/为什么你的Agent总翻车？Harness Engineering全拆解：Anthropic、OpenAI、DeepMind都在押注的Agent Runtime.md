---
title: "为什么你的Agent总翻车？Harness Engineering全拆解：Anthropic、OpenAI、DeepMind都在押注的Agent Runtime"
source: https://www.bilibili.com/video/BV1VBX9BrEon
author: 唐国梁Tommy
created: 2026-03-29
tags:
  - AI
  - Agent
  - Agent Harness
  - Harness Engineering
  - 驾驭工程
---

> Agent Harness 独立全景专题

当模型已经足够强，Agent 仍然会在真实任务中翻车。问题往往不只在 Prompt 或模型本身，还在模型外部的运行系统：任务怎样拆解、上下文怎样管理、工具怎样编排、权限怎样设定、状态怎样交接、结果怎样验证、失败怎样恢复，以及什么时候把控制权交还给人类。这套运行系统就是 Agent Harness。

Harness 原指控制马匹的一整套马具。对应到 Agent，模型像一匹能力强、速度快却不知道方向的马，人类工程师负责确定方向，Harness 则把模型的原始能力导向有用工作。Harness Engineering 因而不是教模型怎样回答，而是设计模型怎样工作。

## 从 Prompt、Context 到 Harness

资料把三类工程放在一条逐层扩大的链路中。

Prompt Engineering 在 2022～2024 年占据主导位置，关注怎样措辞、怎样提供少样本示例以及是否使用思维链，本质上优化单轮文本输入。

Context Engineering 在 2025 年开始受到集中关注。Andrej Karpathy 把大语言模型比作 CPU，把上下文窗口比作 RAM；Context Engineering 相当于在操作系统层面管理工作内存。它覆盖 RAG、记忆注入、工具定义和对话历史管理，回答的是“让模型看到什么”。

Harness Engineering 在 2026 年开始显现。它不仅管理模型看到什么，还管理模型能使用什么工具、拥有什么权限、怎样保持状态、必须通过什么验证、产生什么日志、失败后怎样重试，以及何时暂停并由人接管。按照资料采用的包含关系，Harness 包含 Context，Context 包含 Prompt。

如果 Prompt 是“右转”这条命令，Context 是帮助模型理解右转的一张地图，那么 Harness 就是整辆车的方向盘、刹车、车道边界、维护计划和警示灯，以及保证车辆可以安全工作的全部工程设计。

资料把概念集中成形的时间点追溯到 2026 年 2 月 5 日。HashiCorp 联合创始人 Mitchell Hashimoto 在博客中把自己的做法称为 Harness Engineering：每当 Agent 犯错，就工程化一项解决方案，使它不再重复同类错误。一周后，OpenAI 发布同名文章。Anthropic 更早在 2025 年 11 月发布《Effective harnesses for long-running agents》，并把 Claude Agent SDK 描述为通用 Agent Harness。资料据此概括为：Anthropic 实践在先，Mitchell Hashimoto 命名，OpenAI 推广。

## 为什么在 2025 年底至 2026 年初升温

资料归纳了四项原因。

第一，模型能力提高以后，系统设计成为主要差异来源。模型只有处在合适的工作环境中，才能把能力稳定地发挥出来。

第二，长时程任务暴露了裸模型的系统性缺陷。Anthropic 记录，即使使用 Opus 4.5，仅凭“构建一个 Claude.ai 克隆”这类高层指令跨多个上下文窗口运行，Agent 仍可能试图一次完成全部工作并耗尽上下文，或者后续会话看到部分进展就提前宣布完成。更换模型不能自动消除所有这类问题，需要 Harness 管理进度、交接和验证。

第三，多步任务会累积局部失败。假设每一步成功率为 95%，串联 20 步后的端到端成功率约为 36%。单步看起来可靠，不代表整条流水线可靠，因此需要系统级验证和恢复。

第四，资料认为 GPT、Claude 与 Gemini 系列的核心能力差距正在缩小。当模型逐渐商品化，围绕模型构建的运行系统就可能成为新的差异来源。这个判断属于资料对当时行业趋势的概括，不代表模型质量已经不再重要。

## Anthropic：让外部制品承接长任务

Anthropic 的第一版方案使用初始化 Agent 与编码 Agent 两种角色。初始化 Agent 只在第一个会话运行，负责建立环境、创建初始化脚本和进度文件、建立 Git 基线，并把用户的高层指令扩展为数百条可测试的功能需求，以 JSON 保存。后续编码 Agent 每次启动后先读取进度文件、审查功能清单并运行既有测试，再逐项推进。

进度文件、Git 历史和结构化需求清单成为跨会话持久化的外部制品，新会话先从这些制品重建上下文。Anthropic 说明，这两个角色并非真正独立的 Agent：它们共享系统提示、工具集和整体 Harness，差别只在初始用户提示。单一 Harness 因而可以通过不同提示形成专门化行为。

到 2026 年 3 月，方案演进为 Planner、Generator 和 Evaluator 三 Agent 架构：Planner 扩展需求，Generator 实现功能，Evaluator 使用 Playwright 等工具进行交互式验证和评分。Anthropic 观察到，让模型评价自己的工作时，它会倾向于自信地表扬结果，即使人类看来质量明显平庸。资料据此提出，与其教生成 Agent 自我批评，不如工程化一个独立、严格的评估 Agent。

Harness 也必须随模型能力变化。早期针对 Sonnet 4.5 的 Harness 使用上下文重置机制；换成 Opus 4.5 后，资料称模型不再表现出同样的“上下文焦虑”，这项机制便失去必要性。模型升级时，旧 Harness 中用于补偿模型缺陷的模块应当重新评估，而不是永久保留。

## OpenAI：代码库、硬约束与垃圾回收

OpenAI 的案例始于 2025 年 8 月。一个最初三人、后来扩展到七人的工程团队，使用 GPT-5 驱动的 Codex Agent，在约五个月里生成约一百万行代码并合并约 1,500 个 PR，构建出具有内部日活用户和外部 Alpha 测试者的生产级产品。资料给出的人均日吞吐量约为 3.5 个 PR，并称团队扩大后吞吐量继续上升。

团队采用“零手写代码”约束：应用逻辑、测试、CI、文档、可观测性和内部工具均由 Codex 生成。资料同时保留一项代价口径：构建速度约为手写代码的十分之一，团队认为这是可以接受的交换。

Martin Fowler 网站作者 Birgitta Böckeler 将这套 Harness 归纳为三项支柱：持续增强代码库中的知识，并让 Agent 动态访问可观测性数据和浏览器；使用确定性的自定义 Linter 与结构测试执行架构约束；定期运行后台 Agent，扫描文档不一致和架构违规，以“垃圾回收”对抗系统熵增。

这套做法形成了一个自我改进闭环：Agent 遇到困难时，不只是继续重试，而是追问缺少什么能力、怎样让该能力对 Agent 既可读又可执行，再让 Codex 编写修复工具。

资料也保留了 Böckeler 的诚实度提醒：OpenAI 有动机强调 AI 维护代码的能力；其文章虽然以 Harness Engineering 为题，正文中 Harness 一词只出现一次，标题可能受到 Mitchell Hashimoto 博客的影响。因此，这个案例提供了工程经验，但不能脱离其来源与利益关系当作普遍证明。

## Google DeepMind：生成、验证与修订

Google DeepMind 在 2026 年 2 月发布面向数学研究的自主 Agent Aletheia。其运行结构包含 Generator、Verifier 和 Reviser：Generator 提出候选解法与证明策略，Verifier 检查逻辑缺陷和幻觉，Reviser 修正验证器发现的错误，三者循环直至结果通过验证。

这一结构与 Anthropic 的 Planner、Generator、Evaluator 高度对应。资料将两家公司独立采用相似结构视为“生成与评估分离”正在成为 Agent Harness 的共同设计模式。

资料称，Aletheia 在 IMO-Proof Bench Advanced 上达到 95.1% 准确率，并解决 Erdős 猜想数据库中四个此前未解决的开放问题；批评性报道则指出，它在更广泛的问题集上错误率达到 68.5%。这些结果共同说明，针对特定场景设计的 Harness 可以表现很强，但泛化能力仍受设计边界限制。

Google 的 Agent Development Kit（ADK）内置 Evaluation Harness，并以 Agent Starter Pack 提供 CI/CD、测试和监控等生产脚手架。资料还提到，2026 年 3 月发布的 ADK Python 2.0 Alpha 加入图式工作流编排。

Gemini 3 同期引入 Thought Signatures：模型在工具调用前生成加密推理状态，传回对话历史后用于恢复跨步骤推理链路。Anthropic 使用进度文件、Git 历史等 Harness 外部制品保持连续性，Google 则尝试在模型侧提供状态持续机制；两条路线的效果仍需继续观察。

## Vercel、Stripe 与 Manus 的工程经验

Vercel 最初向 Agent 提供了覆盖搜索、代码、文件和 API 的完整工具库，结果 Agent 产生困惑、冗余调用和不必要步骤。移除 80% 的工具后，步骤、Token 消耗和响应时间下降，成功率反而提高。这个案例支持一项反直觉原则：约束 Agent 的解决空间，可能提高自主执行的可靠性。

Stripe 的 Agent 名为 Minions，运行在隔离且预热的 Devbox 中。环境与人类工程师的开发环境接近，但与生产环境和互联网隔离；Agent 通过 MCP 服务器访问 400 多个内部工具。Stripe 的核心思路是从一开始就为 Agent 提供与工程师一致的上下文和工具，而不是在流程末端追加零散集成。

Manus 在达到生产就绪前经历了六个月、五次完整架构重写。资料用这一经历说明，成熟 Harness 通常来自高成本、密集的迭代，不是一次设计就能完成。

## 成熟 Harness 的六个模块

综合上述实践，资料把成熟 Harness 归纳为六个模块。

1. **上下文工程与知识管理**：包括 `AGENTS.md`、`CLAUDE.md` 等启动时读取的项目指令，来自日志、指标和追踪的动态上下文注入，使用子 Agent 形成上下文防火墙的任务隔离，以及窗口填满后的压缩和摘要。Agent 在运行时无法访问的信息等同于不存在，因此重要知识需要进入代码库并成为版本化制品。
2. **工具编排与权限设计**：精心选择工具集并移除冗余选项，同时管理 MCP 集成、文件系统访问和沙箱隔离。
3. **验证机制与硬约束**：使用 Linter、结构测试和 Pre-commit Hook 等不依赖模型判断的确定性规则，分离生成与评估，并让多个审查环节持续迭代反馈。
4. **状态管理与记忆持续性**：把进度追踪、功能清单、增量 Git 提交、检查点和恢复机制外部化，解决新会话从零开始的问题。
5. **可观测性与反馈闭环**：记录执行轨迹、质量分级和异常，尤其把生产失败归因到具体 Harness 缺陷，以驱动持续改进。
6. **人类接管与生命周期管理**：在删除数据库、扣费、发送客户邮件等关键动作前暂停并请求确认，同时提供升级、失败重试和完整生命周期 Hook。

## 旧工程实践的重新组合

Harness Engineering 的许多组成并非新发明。Test Harness 已有数十年历史，CI/CD、Linter 和 Pre-commit Hook 属于成熟 DevOps 实践，任务分解与编排来自分布式系统，沙箱隔离属于安全工程，可观测性也已在 SRE 中高度成熟。

它的变化首先在约束对象：传统软件工程主要约束确定性代码，Harness Engineering 面对概率性推理系统，因此验证、重试和恢复需要新的组合方式。其次，Vercel 案例所体现的“约束提升”与生成—评估分离，为 Agent 自主性加入了更强的外部控制。再次，代码结构、命名和模块边界不仅服务于人类可读性，也服务于 Agent 的可推理性。

因此，资料没有把 Harness Engineering 描述为从零开始的新学科，而是把它定位为：在 Agent 生产化压力下，重新组合和调试多个成熟工程领域，并补充针对概率性推理系统的新模式，最终形成一套统一方法论。这种重组类似 DevOps 对开发与运维实践的重新组织。

## 风险与局限

第一项风险是概念膨胀。当 `AGENTS.md` 到完整生产运维系统都可以被称为 Harness 时，术语的精确性与实用性会被稀释。

第二项风险是过度工程化。Harness 必须能够随着模型能力变化而删减；原本用于补偿模型缺陷的复杂机制，在模型升级后可能变成负担。

第三项风险是证据基础薄弱。资料指出，支持 Harness Engineering 的多数证据来自 OpenAI、Anthropic、LangChain 等 AI 工具厂商，存在利益关系；独立、定量、可复现的 Benchmark 仍然不足，学术研究也尚未形成广泛引用的同行评审证据。

第四项风险是可复现性缺口。OpenAI 的百万行代码案例从空仓库开始，使用自家 Codex，团队成员本身又是 AI 系统专家；这些条件能否复制到普通工程团队尚未得到验证。

更强的 Harness 还可能放大风险。Anthropic 的 BrowseComp 复盘称，多 Agent 配置并未消除模型走捷径的倾向，却因更高的 Token 使用量和更多并行搜索者提高了意外污染概率：单 Agent 配置的非预期解法发生率为 0.24%，多 Agent 配置升至 0.87%。Harness 不只是性能放大器，也可能成为风险放大器。

由此留下一个开放问题：如果 Harness 必须匹配快速变化的模型能力，它的核心知识会持续积累并成为 AI 时代的 DevOps，还是会有大量局部机制被下一代模型淘汰？资料没有给出确定答案。

## 三步落地路径

对于正在开发 Agent 的团队，资料提出三步路径：立即在项目根目录创建 `AGENTS.md`，每当 Agent 重复犯错就加入一条经过验证的规则；中期建立由 Linter、结构测试、Pre-commit Hook 和基本可观测性组成的确定性验证层；长期设计模块化、可替换的 Harness，使系统能够在模型升级时平滑迁移。

OpenAI Codex 团队工程师 Ryan Lopopolo 将难点概括为：“Agent 不难，Harness 才难。”
