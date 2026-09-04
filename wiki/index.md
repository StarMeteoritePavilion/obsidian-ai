---
title: AI 知识索引
updated: 2026-09-04
tags:
  - AI
  - 索引
---

# AI 知识索引

本知识库收录 57 份资料摘要和 7 篇跨资料综合。AI 应用工程沿 Prompt、Context、Loop、Evaluation 四个阶段组织，并收录 AI Agent 与驾驭工程；模型相关资料按模型架构、大语言与多模态原理、训练与后训练、推理优化组织。原始事实保存在 `raw/sources/`，本目录负责摘要、关联、冲突记录与综合判断。

维护历史见 [[wiki/log|维护日志]]。

## 总体主线

| 阶段 | 核心问题 | 主要产物 | 综合入口 | 进入下一阶段的条件 |
| --- | --- | --- | --- | --- |
| Prompt | 当前任务怎样表达 | 指令、示例、推理方法、输出契约 | [[wiki/syntheses/提示词工程：从单轮指令到生产规范]] | 单条指令无法管理历史、知识和工具 |
| Context | 当前轮次让模型看到什么 | 检索、压缩、分层、隔离和路由 | [[wiki/syntheses/上下文工程：有限窗口中的信息治理]] | 单轮信息需要跨轮触发、回写和停止 |
| Loop | 系统跨轮次怎样继续 | 调度、持久状态、独立评判和停止机制 | [[wiki/syntheses/循环工程：从逐轮操作到外部调度]] | 循环需要业务标准证明能否继续或上线 |
| Evaluation | 怎样证明系统满足业务要求 | 数据集、评分器、指标、治理和质量门禁 | [[wiki/syntheses/评估工程：从通用基准到业务质量门]] | 评估结果回流前三层，形成改进闭环 |

四个阶段不是互相替代，而是逐层扩大工程对象：Prompt 管表达，Context 管信息，Loop 管决策流，Evaluation 管质量证据。Harness 横切前三层，为工具、权限、安全、隔离和恢复提供共同环境。

## 按问题进入

| 当前问题 | 优先阅读 |
| --- | --- |
| 指令、格式或示例不稳定 | [[wiki/syntheses/提示词工程：从单轮指令到生产规范]] |
| 需要区分 Prompt、Agent、Function Calling 与 MCP | [[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP]] |
| 需要理解 Prompt 怎样扩展为 Agent 上下文治理 | [[wiki/sources/上下文工程：从 Prompt 到 Agent 上下文治理]] |
| 文档存在却答不到、历史过长或工具过多 | [[wiki/syntheses/上下文工程：有限窗口中的信息治理]] |
| 需要理解个人知识库的基础向量 RAG 链路与局限 | [[wiki/sources/上下文工程：RAG 个人知识库基础架构]] |
| 需要用实体关系与分层摘要同时支持局部和全局检索 | [[wiki/sources/上下文工程：GraphRAG 从知识图谱到分层检索]] |
| 需要在固定窗口预算下分配 RAG 文档、示例与迭代次数 | [[wiki/sources/上下文工程：DRAG 与 IterDRAG 推理扩展]] |
| 需要定时运行、跨轮接力、独立验证或停止条件 | [[wiki/syntheses/循环工程：从逐轮操作到外部调度]] |
| 不知道系统是否真的变好、能否上线或是否发生回退 | [[wiki/syntheses/评估工程：从通用基准到业务质量门]] |
| 需要比较 Agent 框架或确定技术选型 | [[wiki/sources/AI Agent 框架选型：十大框架与五大范式]] |
| 需要判断 SFT 后是否使用 RL，或怎样选择 RLHF、DPO、GRPO、RLVR | [[wiki/syntheses/大模型后训练：从模仿到行为选择]] |
| 需要保存、复制和恢复长时程 Agent 的沙箱环境 | [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV]] |
| 需要比较长序列记忆、召回与计算成本 | [[wiki/sources/长序列建模：Memory Caching]] |
| 需要理解 Transformer 的 Encoder—Decoder、Decoder-only 与 Encoder-only 分支 | [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]] |
| 需要理解 Linear、Weight、Bias、Activation、FFN 与 MLP | [[wiki/sources/模型架构：Linear、Activation 与 MLP]] |
| 需要理解多头注意力、QKV、因果 Mask 与 Softmax | [[wiki/sources/模型架构：多头注意力与 QKV]] |
| 需要理解标准残差、Attention Residuals 与跨层选择性聚合 | [[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]] |
| 需要理解 MoE、Router、专家激活与总参数／激活参数差异 | [[wiki/sources/模型架构：MoE 稀疏专家路由]] |
| 需要理解块级稀疏注意力与长上下文计算 | [[wiki/sources/模型架构：MoBA 混合块注意力]] |
| 需要理解 Tokenization、隐藏表示或 Latent Reasoning | [[wiki/sources/模型原理：Token Space 与 Latent Space]] |
| 需要判断可见思维链是否忠实，以及任务、长度与格式变化怎样影响 CoT | [[wiki/sources/大语言模型：思维链的模式匹配与泛化边界]] |
| 需要理解 Kimi K2 Thinking 的 MoE、MuonClip、Agent 训练与 INT4 | [[wiki/sources/大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练]] |
| 需要理解文本与图像怎样交错推理 | [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]] |
| 需要梳理多模态模型的架构、数据、推理、CMR 与 RAG | [[wiki/sources/多模态模型：架构、数据、推理与检索]] |
| 需要理解视觉推理中的指代漂移、框与点及视觉 Token 压缩 | [[wiki/sources/多模态推理：视觉原语与 Reference Gap]] |
| 需要理解视觉原语的数据过滤、专家训练、OPD 与密集奖励 | [[wiki/sources/多模态推理：视觉原语的数据、训练与奖励]] |
| 需要在保持输出分布的前提下提高模型生成速度 | [[wiki/sources/模型推理优化：DSpark 投机解码]] |
| 需要估算 API Token、长上下文、缓存与批处理的任务成本 | [[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]] |
| 需要让短任务训练迁移到长任务或新领域 | [[wiki/sources/大模型后训练：RLM Harness 组合泛化]] |
| 需要让 Agent 冻结权重并从运行时反馈更新记忆 | [[wiki/sources/Agent 记忆：MemRL 运行时强化学习]] |
| 需要理解 Agent 如何压缩环境、判断动作后果或选择检查器与模拟器 | [[wiki/sources/Agent 世界模型：服务于行动的选择性压缩]] |

## 资料摘要

### 提示词工程

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/提示词工程：第一期 提示词工程入门]] | 提示词四要素、零样本提示、指令微调和基础能力边界 |
| [[wiki/sources/提示词工程：第二期 少样本提示]] | 上下文学习、示例选择、格式一致性和多步推理限制 |
| [[wiki/sources/提示词工程：第三期 让 AI 先想再说]] | CoT、零样本 CoT、Auto-CoT、ToT 的机制与代价 |
| [[wiki/sources/提示词工程：第四期 多步编排]] | 自我一致性、链式提示、结构化中间结果和成本边界 |
| [[wiki/sources/提示词工程：第五期 知识增强与工具调用]] | 生成知识、程序化推理、工具调用和 Agent 工作流 |
| [[wiki/sources/提示词工程：第六期 Anthropic 工程规范]] | XML、system prompt、证据约束、长文排序和版本边界 |

### 上下文工程

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/上下文工程：第一期 从 Prompt 到 Context]] | 提示、上下文、框架、记忆工程的边界与六类信息 |
| [[wiki/sources/上下文工程：第二期 窗口与 Token]] | 窗口容量、Token 预算、KV 缓存、注意力和中间遗忘 |
| [[wiki/sources/上下文工程：第三期 原则、策略、评估]] | 注意力预算、写入选择压缩隔离和三层评估 |
| [[wiki/sources/上下文工程：从 Prompt 到 Agent 上下文治理]] | 两类 Prompt、工具循环膨胀、任务笔记、历史修剪、摘要与长返回外移 |
| [[wiki/sources/上下文工程：RAG 个人知识库基础架构]] | 分块、Embedding、向量存储、相似片段检索及局部检索边界 |
| [[wiki/sources/上下文工程：第四期 RAG 检索增强生成]] | 原始 RAG、分块、混合检索、查询增强和可追溯性 |
| [[wiki/sources/上下文工程：第五期 上下文工程压缩]] | 四类压缩、锚点保护、Compaction 和 KV 缓存优化 |
| [[wiki/sources/上下文工程：第六期 结构化与隔离]] | 分隔符、XML、JSON、任务隔离、沙箱和子上下文 |
| [[wiki/sources/上下文工程：第七期 上下文是怎么坏掉的]] | 毒化、分心、混淆、冲突和 GraphRAG |
| [[wiki/sources/上下文工程：第八期 2026 生产实践]] | Skills、混合压缩、路由、自主检索和工具管理 |
| [[wiki/sources/上下文工程：DRAG 与 IterDRAG 推理扩展]] | DRAG 演示、IterDRAG 迭代检索、固定预算优化与推理扩展边界 |
| [[wiki/sources/上下文工程：GraphRAG 从知识图谱到分层检索]] | 实体关系抽取、来源映射、Leiden 社区分层及 Local Search 与 Global Search |

### 循环工程

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/循环工程：先导篇 从 Prompt 到 Loop]] | 系列定义、路线、工程账目和人的位置迁移 |
| [[wiki/sources/循环工程：第一期 什么是 Loop Engineering？]] | 三层递进、Harness 外壳、循环动作和失败放大 |
| [[wiki/sources/循环工程：第二期 三大流派与四笔代价]] | 三种自治哲学、执行视界、盈亏平衡和四笔代价 |
| [[wiki/sources/循环工程：第三期 Loop 该怎么搭？]] | 五个动作、六个组件、独立评判器和 Backpressure |
| [[wiki/sources/循环工程：第四期 搭好 Loop ≠ 能上线？]] | 共用基础设施、五步路线和六项上线检查 |

### 评估工程

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/评估工程：第一期 排行榜遥遥领先，用起来怎么各种拉胯？]] | Benchmark 边界、黄金数据、评分方式和 CI/CD 路线 |
| [[wiki/sources/评估工程：第二期 模型答对了，代码却判了错？]] | 精确匹配、集合匹配、结构化输出和失败分析闭环 |
| [[wiki/sources/评估工程：第三期 让 AI 评价 AI，为什么能成立？]] | 领域化 LLM 裁判、五种偏见、多裁判和人工校准 |
| [[wiki/sources/评估工程：第四期 评估场景，为什么不需要瑞士军刀？]] | SLM 裁判、LoRA 多指标、蒸馏量化和分层路由 |
| [[wiki/sources/评估工程：第五期 能跑的评估和能放心用的评估，差在哪一层？]] | Ground Truth、SME 精炼、策略采样、双指标和治理 |
| [[wiki/sources/评估工程：第六期 Agent 评估为什么比 LLM 评估难一个数量级？]] | Agent 状态评估、Capability、Regression、双指标和分型评分 |
| [[wiki/sources/评估工程：第七期 从事后评估到生产护栏，差的是挡住还是知道？]] | 五组件、输入与输出护栏、误报级联、影子模式和渐进上线 |
| [[wiki/sources/评估工程：第八期 AI 评估的最后一公里到底长什么样？]] | 合同驱动、Source-to-Claim、组合边界、输出卫生和企业治理 |

### 驾驭工程

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/驾驭工程：系列完结，下一步该往哪走？]] | Harness 工程系列收束、作者定义与观测、优化、安全三个后续方向 |
| [[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness]] | Processor 生命周期、AEGIS、确定性闸门、变体隔离与 Cross-Harness GRPO |

### AI Agent

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP]] | User Prompt、System Prompt、Agent、工具调用接口与 MCP 服务边界 |
| [[wiki/sources/AI Agent 框架选型：十大框架与五大范式]] | MCP、A2A、五种架构范式、十大框架地图、四步选型与隐性成本 |
| [[wiki/sources/Agent 记忆：MemRL 运行时强化学习]] | 冻结模型权重，以语义粗筛、Q-value 精排和环境反馈更新实现运行时情景记忆学习 |
| [[wiki/sources/Agent 世界模型：服务于行动的选择性压缩]] | 环境、判断与知识三层结构，检查器和学习型世界模型两条路线，以及记忆过期与验证边界 |

### 模型训练与后训练

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/大模型后训练：强化学习如何选择反馈与算法]] | RLHF、RLAIF、RLVR、DPO、GRPO 的反馈条件，Agent RL 五个工程问题与奖励黑客 |
| [[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习]] | 轨迹到技能的蒸馏、冷启动 SFT、技能增强 RL、验证驱动进化及工程限制 |
| [[wiki/sources/大模型后训练：RLM Harness 组合泛化]] | 局部分布内、Context 卸载、程序化子调用及跨长度与跨领域组合泛化 |
| [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV]] | Partial Rollout、microVM 沙箱、暂停恢复、环境分岔、增量检查点与 Off-Policy 边界 |

### 模型架构

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]] | 原始 Encoder—Decoder 翻译架构、逐 Token 生成、监督与自监督训练，以及 GPT 与 BERT 分支 |
| [[wiki/sources/模型架构：Linear、Activation 与 MLP]] | 线性映射、权重与偏置、梯度下降、ReLU／Sigmoid／tanh／GELU、FFN 与示例 MLP |
| [[wiki/sources/模型架构：多头注意力与 QKV]] | Embedding 到 QKV 投影、缩放点积、因果 Mask、Softmax、Value 加权与多头拼接 |
| [[wiki/sources/模型架构：MoBA 混合块注意力]] | 块级稀疏路由、可变长度 FlashAttention、online Softmax 及长上下文效率边界 |
| [[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]] | 标准残差的数值膨胀与信息稀释、pseudo-query、Full AttnRes、Block AttnRes 及资源取舍 |
| [[wiki/sources/模型架构：MoE 稀疏专家路由]] | FFN 与 Dense 模型、Router、路由专家、共享专家、激活参数及 DeepSeekMoE 对比边界 |
| [[wiki/sources/长序列建模：Memory Caching]] | 分段隐状态检查点、四种聚合策略及 RNN 长程召回与计算成本的连续权衡 |

### 模型推理优化

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/模型推理优化：DSpark 投机解码]] | 首 Token 容量、半自回归草稿、置信度调度、校准、无损早停及 DeepSeek-V4 线上结果 |
| [[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]] | Prefill 与 Decode、Reasoning Token、KV Cache、Prompt Caching、Batch、任务总成本与 Token FinOps |

### 模型原理

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/模型原理：Token Space 与 Latent Space]] | Token 到隐藏表示再回到 Token 的生成路径、分词机制、模型可解释性与 Latent Reasoning |
| [[wiki/sources/大语言模型：思维链的模式匹配与泛化边界]] | DataAlchemy、任务／长度／格式泛化、可见 CoT 与答案不一致及训练分布边界 |
| [[wiki/sources/大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练]] | 1.04T MoE、MuonClip、Data Rephrasing、Agent SFT/RL、原生 INT4 与长程工具调用 |
| [[wiki/sources/多模态推理：ThinkMorph 交错思维链]] | 文本规划与视觉操作交替推进、三种涌现能力、测试时扩展及适用边界 |
| [[wiki/sources/多模态模型：架构、数据、推理与检索]] | 视觉编码、模态接口、数据工程、MCoT、跨模态检索与多模态 RAG 的完整链路 |
| [[wiki/sources/多模态推理：视觉原语与 Reference Gap]] | Perception Gap 与 Reference Gap、框和点作为推理变量、类 LLaVA 架构及 7056× 工程压缩链路 |
| [[wiki/sources/多模态推理：视觉原语的数据、训练与奖励]] | 两阶段数据过滤、框点专家训练、Unified RFT、OPD、三层奖励及拓扑评测边界 |

## 模型工程综合

| 页面 | 内容 |
| --- | --- |
| [[wiki/syntheses/大模型后训练：从模仿到行为选择]] | 从 SFT 模仿到偏好与强化学习的路线选择、轨迹训练、技能蒸馏和独立评测 |
| [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]] | 离散 Token、内部连续表示与显式多模态交错思维的职责、成本和审计边界 |
| [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]] | Harness 的信息、行动、控制、安全、观测与训练职责，以及静态外壳到可进化对象的边界 |

## 当前来源边界

- 资料中的模型表现、价格、延迟、Token、准确率和硬件数字保留来源属性，不能脱离实验或案例条件外推。
- Claude、Codex、MCP 和其他工具的命令及 API 行为具有版本时效性，实际使用前需核对对应官方文档。
- AI Agent 基础资料关于 Function Calling 厂商差异与开源模型支持情况的判断对应 2025 年 5 月；模型负责提出工具调用，Agent 或应用负责执行工具这一职责边界不随具体接口名称改变。
- Prompt 到上下文治理资料关于网页产品内置推理、消息角色和 Agent 实现的描述对应 2025 年 8 月；实际应用需要按当前 API、模型和状态管理方式重新核对。
- 评估工程系列共八期，当前知识库已经完整收录；第八期以合同驱动架构收束从代码评分到企业治理的演进。
- 驾驭工程收尾篇采用作者的宽泛 Harness 定义；其术语范围与本库此前“Harness 作为 Prompt、Context 与 Loop 共同外壳”的来源表述并存，不合并为统一定义。
- Agent 框架资料反映 2026 年 3 月的版本、生态与商业模式；实际选型前需要重新核对官方文档。视频内 Dify Star 数存在旁白与画面差异，Agno 也未获得与其余框架同等篇幅的分析。
- 当前大模型后训练综合由强化学习路线总览、Kimi K2 Thinking、SKILLRL、RLM Harness、HarnessX、MemRL 与 Kimi K3 AgentENV 共同支撑；其中数字来自各自模型、任务、论文和基础设施设置，不构成统一数据集上的效果排名或其他 Agent 的收益保证。
- Memory Caching 的实验数字来自资料转述的特定模型、规模和任务；其模型架构层记忆机制不等同于应用层上下文治理。
- MoBA 的复杂度、百万 Token 召回及约 6.5 倍计算时间改进来自资料转述的 Llama-8B-1M、块划分、Top-k 与对应硬件设置；不能外推为其他模型和部署的保证。
- 多头注意力资料中的“语法表”“需求表”“内容矩阵”和可读 Attention Head 均为教学类比，不代表真实隐向量维度可以逐项命名；1600、7168 与商业模型上万维的表述保留来源对应模型和推测边界。
- Linear 与 MLP 资料中的 GPT-2 XL `1600→50257`、示例 MLP `1→128→256→1`、33537 个参数和 2000 条数据来自对应模型与教学实验；“从0开始一起学大模型”合集顺序不等于视频正式标注的期数。
- Attention Residuals 资料只讲解机制与开销，没有介绍论文实验成绩；人物识别层与关系层属于教学类比，不能据此认定真实模型各层具有可直接命名的固定分工。
- MoE 资料中的 256 个路由专家与每次选择 8 个专家属于 DeepSeek-V3 配置；144.6B／22.2B、67.4B、585.6T／2057.5T 和 Pile Loss 来自 DeepSeekMoE 对比设置，不能直接外推为其他模型的速度或 Token 价格。
- Tokenization、SuperBPE、T-Free、SAE、Coconut 和 Soft Thinking 的性能或节省数字来自资料转述的对应研究设置；不能据此外推到其他模型、语言和任务。Latent 表示也不构成模型具有意识或主观体验的证据。
- Kimi K2 Thinking 的架构参数、15.5T Token 稳定训练、INT4 约 2 倍生成速度、200～300 次连续工具调用和评测数字来自资料对应的模型、硬件、工具与预算设置；不能外推为其他部署的收益或长程可靠性保证。
- ThinkMorph 的实验数字来自 BAGEL-7B、24,990 条训练轨迹及对应基准；模式切换的 5.3% 在讲解文字与论文图注中分别归于 MMVP 和 Chart Refocus，本库保留该来源冲突。
- 多模态技术地图中的架构提升、训练数据、推理成绩、检索指标和延迟来自资料转述的不同论文设置；不能合并为统一排行榜或外推为其他模型、任务和部署的普遍规律。
- 视觉原语资料的 7,056× 是原始像素数与视觉 KV Cache 条目数之间的工程比值；90 个缓存条目与 77.2 平均分来自论文指定分辨率和七项选定评测，不代表模型整体能力。
- 视觉原语下集的 4,000 万样本、66.9% 迷宫导航和 56.7% 路径追踪结果来自报告的特定数据、低推理预算与评测协议；报告没有标准消融表，不能分别量化视觉原语、框点分训、OPD 和 CSA 的贡献。
- RLM Harness 的组合泛化结果只验证了一个 30B 底座和适合切块的任务；训练样本耗时为直接训练的 1.5～3.0 倍，不能外推为所有模型或高度耦合任务的通用结论。
- DSpark 的接受长度、草稿接受率和 60%～85% 单用户生成速度提升来自资料转述的对应实验及 DeepSeek-V4 线上负载；不能外推到其他模型、硬件、批量或流量结构。
- Token API 资料中的价差、输入输出倍率、长上下文分档、缓存节省上限和 Batch 折扣反映视频发布时的平台规则概括；实际采购需核对具体模型、区域、服务等级和当前官方价格页。
- MemRL 的 3.8 个百分点平均提升、探索密集型任务 6.2 个百分点提升与约零额外推理成本来自资料转述的四项实验；不能外推为其他 Agent 的效果或成本保证。
- RAG 个人知识库资料中的 1536 维与 3072 维分别对应 `text-embedding-3-small` 和 `text-embedding-3-large`；Pinecone、ChromaDB 及 PostgreSQL 配合 pgvector 只是来源列举的存储选择，不构成固定架构要求。
- DRAG 与 IterDRAG 的平均准确率、CoT 对比和参数热图来自资料转述的 Gemini 1.5 Flash、四项基准及对应配置空间；不能外推为其他 RAG 系统的固定参数或收益保证。
- Kimi K3 AgentENV 的沙箱、镜像、最低检查点与恢复延迟、98% 等待占比和最高 6.5 倍内存超配来自对应训练与评估工作负载；不能外推为其他集群的容量、延迟或资源利用保证。
- Agent 世界模型资料中的 78%、90%、约 4%、71% 和 94% 来自对应下棋与易犯规游戏设置；合成世界训练略微超过真实环境训练也只属于 Qwen-AgentWorld 的对应实验，不能外推为所有 Agent 或训练环境的收益。
