---
title: AI 知识索引
updated: 2026-09-04
tags:
  - AI
  - 索引
---

# AI 知识索引

本知识库收录 77 份资料摘要和 11 篇跨资料综合。AI 应用工程沿 Prompt、Context、Loop、Evaluation 四个阶段组织，并收录 AI Agent 与驾驭工程；模型相关资料按模型架构、大语言与多模态原理、训练与后训练、推理优化、计算基础设施组织。原始事实保存在 `raw/sources/`，本目录负责摘要、关联、冲突记录与综合判断。

维护历史见 [[wiki/log|维护日志]]。

## 总体主线

| 阶段 | 核心问题 | 主要产物 | 综合入口 | 进入下一阶段的条件 |
| --- | --- | --- | --- | --- |
| Prompt | 当前任务怎样表达 | 指令、示例、推理方法、输出契约 | [[wiki/syntheses/提示词工程：从单轮指令到生产规范]] | 单条指令无法管理历史、知识和工具 |
| Context | 当前轮次让模型看到什么 | 检索、压缩、分层、隔离和路由 | [[wiki/syntheses/上下文工程：有限窗口中的信息治理]] | 单轮信息需要跨轮触发、回写和停止 |
| Loop | 系统跨轮次怎样继续 | 调度、持久状态、独立评判和停止机制 | [[wiki/syntheses/循环工程：从逐轮操作到外部调度]] | 循环需要业务标准证明能否继续或上线 |
| Evaluation | 怎样证明系统满足业务要求 | 数据集、评分器、指标、治理和质量门禁 | [[wiki/syntheses/评估工程：从通用基准到业务质量门]] | 评估结果回流前三层，形成改进闭环 |

四个阶段不是互相替代，而是逐层扩大工程对象：Prompt 管表达，Context 管信息，Loop 管决策流，Evaluation 管质量证据。Harness 横切前三层，为工具、权限、安全、隔离和恢复提供共同环境。

## 综合导航

| 领域 | 综合入口 | 主要边界 |
| --- | --- | --- |
| 提示词工程 | [[wiki/syntheses/提示词工程：从单轮指令到生产规范]] | 推理时表达，不修改模型权重 |
| 上下文工程 | [[wiki/syntheses/上下文工程：有限窗口中的信息治理]] | 管理进入当前窗口的信息 |
| 循环工程 | [[wiki/syntheses/循环工程：从逐轮操作到外部调度]] | 管理跨步骤与跨轮调度 |
| 评估工程 | [[wiki/syntheses/评估工程：从通用基准到业务质量门]] | 提供独立质量证据与放行标准 |
| Agent Harness | [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]] | 管理模型与真实执行之间的运行系统 |
| AI Agent | [[wiki/syntheses/AI Agent：从工具调用到可信行动]] | 串联工具、状态、世界模型、验证与恢复 |
| 大模型后训练 | [[wiki/syntheses/大模型后训练：从模仿到行为选择]] | 区分示范、偏好、奖励与轨迹学习 |
| 模型推理 | [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]] | 区分表示、计算、执行与服务成本 |
| 长上下文架构 | [[wiki/syntheses/长上下文模型架构：共享、筛选、压缩与可增长记忆]] | 对照模型内部共享、筛选、压缩与记忆 |
| 训练稳定性 | [[wiki/syntheses/深层模型训练稳定性：残差、更新与路由]] | 区分梯度、残差、矩阵、路由与策略失稳 |
| 多模态推理 | [[wiki/syntheses/多模态推理闭环：感知、指代、操作与验证]] | 串联视觉编码、指代、操作和验证 |

## 按问题进入

| 当前问题 | 优先阅读 |
| --- | --- |
| 指令、格式或示例不稳定 | [[wiki/syntheses/提示词工程：从单轮指令到生产规范]] |
| 需要区分 Prompt、Agent、Function Calling 与 MCP | [[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP]] |
| 需要用 Pydantic AI 理解工具注册、同步调用与消息历史 | [[wiki/sources/AI Agent 实践：Pydantic AI 工具调用与消息历史]] |
| 需要理解 Prompt 怎样扩展为 Agent 上下文治理 | [[wiki/sources/上下文工程：从 Prompt 到 Agent 上下文治理]] |
| 文档存在却答不到、历史过长或工具过多 | [[wiki/syntheses/上下文工程：有限窗口中的信息治理]] |
| 需要理解个人知识库的基础向量 RAG 链路与局限 | [[wiki/sources/上下文工程：RAG 个人知识库基础架构]] |
| 需要用实体关系与分层摘要同时支持局部和全局检索 | [[wiki/sources/上下文工程：GraphRAG 从知识图谱到分层检索]] |
| 需要把 RAG 查询结果沉淀为可增量维护的长期知识资产 | [[wiki/sources/上下文工程：LLM Wiki 的摄取时编译与知识治理]] |
| 需要在固定窗口预算下分配 RAG 文档、示例与迭代次数 | [[wiki/sources/上下文工程：DRAG 与 IterDRAG 推理扩展]] |
| 需要定时运行、跨轮接力、独立验证或停止条件 | [[wiki/syntheses/循环工程：从逐轮操作到外部调度]] |
| 不知道系统是否真的变好、能否上线或是否发生回退 | [[wiki/syntheses/评估工程：从通用基准到业务质量门]] |
| 需要检验 Agent 在压力、诱惑和规则漏洞下是否仍然安全 | [[wiki/sources/Agent 安全评估：AutoControl Arena 与对齐幻觉]] |
| 需要比较 Agent 框架或确定技术选型 | [[wiki/sources/AI Agent 框架选型：十大框架与五大范式]] |
| 需要从工具、状态、检查器、世界模型和恢复理解 Agent 可信执行 | [[wiki/syntheses/AI Agent：从工具调用到可信行动]] |
| 需要区分 Prompt、Context 与 Harness 的职责和重叠边界 | [[wiki/sources/驾驭工程：Prompt、Context 与 Harness 的边界]] |
| 需要理解 Harness 的行业案例、六个模块、工程边界与风险 | [[wiki/sources/驾驭工程：Harness Engineering 运行系统全景]] |
| 需要理解 Claude Code 的 ReAct 循环、压缩恢复、多 Agent、记忆与纵深安全 | [[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]] |
| 需要判断 SFT 后是否使用 RL，或怎样选择 RLHF、DPO、GRPO、RLVR | [[wiki/syntheses/大模型后训练：从模仿到行为选择]] |
| 需要理解 Agentic RL 的训练崩塌、重要性采样裁剪、步骤级优势与动态采样 | [[wiki/sources/大模型后训练：ARLArena 与 SAMPO 稳定 Agentic RL]] |
| 需要区分预训练、SFT、全量微调与 LoRA，或理解低秩适配的显存构成 | [[wiki/sources/大模型微调：LoRA 低秩适配]] |
| 需要保存、复制和恢复长时程 Agent 的沙箱环境 | [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV]] |
| 需要比较长序列记忆、召回与计算成本 | [[wiki/sources/长序列建模：Memory Caching]] |
| 需要系统比较 GQA、稀疏注意力、压缩注意力与 Memory Caching | [[wiki/syntheses/长上下文模型架构：共享、筛选、压缩与可增长记忆]] |
| 需要理解 Transformer 的 Encoder—Decoder、Decoder-only 与 Encoder-only 分支 | [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]] |
| 需要理解 Linear、Weight、Bias、Activation、FFN 与 MLP | [[wiki/sources/模型架构：Linear、Activation 与 MLP]] |
| 需要用 PyTorch 理解张量形状、批量输入、MNIST 推理与 Softmax 维度 | [[wiki/sources/模型架构：PyTorch 手写数字识别]] |
| 需要理解损失函数、梯度下降、学习率、Batch Size 与 MSE | [[wiki/sources/模型训练：梯度下降与均方误差]] |
| 需要用 PyTorch 串联 Dataset、DataLoader、CrossEntropyLoss、反向传播和 Optimizer | [[wiki/sources/模型训练：PyTorch 手写数字识别实战]] |
| 需要理解多头注意力、QKV、因果 Mask 与 Softmax | [[wiki/sources/模型架构：多头注意力与 QKV]] |
| 需要理解标准残差、Attention Residuals 与跨层选择性聚合 | [[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]] |
| 需要理解 MoE、Router、专家激活与总参数／激活参数差异 | [[wiki/sources/模型架构：MoE 稀疏专家路由]] |
| 需要理解 Engram 怎样以参数化 N-gram 查找补充 MoE 稀疏计算 | [[wiki/sources/模型架构：Engram 参数化记忆查找]] |
| 需要理解块级稀疏注意力与长上下文计算 | [[wiki/sources/模型架构：MoBA 混合块注意力]] |
| 需要区分 GQA 的 KV Cache 压缩与 DSA／MSA 的稀疏注意力筛选 | [[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]] |
| 需要理解 DeepSeek V4 怎样组合 CSA、HCA、mHC、Muon、Anticipatory Routing 与 OPD | [[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性]] |
| 需要区分残差、矩阵更新、注意力、路由与策略优化的稳定性问题 | [[wiki/syntheses/深层模型训练稳定性：残差、更新与路由]] |
| 需要区分 Tokenizer、Token Embedding 与 RAG Embedding | [[wiki/sources/大语言模型：Token 与两类 Embedding]] |
| 需要理解 Tokenization、隐藏表示或 Latent Reasoning | [[wiki/sources/模型原理：Token Space 与 Latent Space]] |
| 需要判断可见思维链是否忠实，以及任务、长度与格式变化怎样影响 CoT | [[wiki/sources/大语言模型：思维链的模式匹配与泛化边界]] |
| 需要理解 Kimi K2 Thinking 的 MoE、MuonClip、Agent 训练与 INT4 | [[wiki/sources/大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练]] |
| 需要理解 Qwen 3.5 的 MoE、混合注意力、原生多模态与 Cline 工作流 | [[wiki/sources/大语言模型：Qwen 3.5 的 MoE、混合注意力与应用演示]] |
| 需要理解 Claude Fable 5 与 Mythos 5 的能力关系、分层开放和 Agent 系统安全 | [[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放]] |
| 需要理解文本与图像怎样交错推理 | [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]] |
| 需要串联多模态感知、指代、视觉操作与程序化验证 | [[wiki/syntheses/多模态推理闭环：感知、指代、操作与验证]] |
| 需要理解图片怎样经图块、局部特征和 Transformer 编码器进入多模态模型 | [[wiki/sources/多模态模型：ViT 图像分块与编码]] |
| 需要理解 DLSS／FSR 怎样利用低分辨率渲染、历史帧和运动信息重建高清画面 | [[wiki/sources/图像重建：DLSS 与 FSR 的时序超分辨率]] |
| 需要梳理多模态模型的架构、数据、推理、CMR 与 RAG | [[wiki/sources/多模态模型：架构、数据、推理与检索]] |
| 需要理解视觉推理中的指代漂移、框与点及视觉 Token 压缩 | [[wiki/sources/多模态推理：视觉原语与 Reference Gap]] |
| 需要理解视觉原语的数据过滤、专家训练、OPD 与密集奖励 | [[wiki/sources/多模态推理：视觉原语的数据、训练与奖励]] |
| 需要在保持输出分布的前提下提高模型生成速度 | [[wiki/sources/模型推理优化：DSpark 投机解码]] |
| 需要理解 CPU、HBM、计算核心、卡间互联与软件生态怎样共同限制 AI 推理和训练 | [[wiki/sources/AI 计算硬件：内存带宽、互联与软件生态]] |
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
| [[wiki/sources/上下文工程：LLM Wiki 的摄取时编译与知识治理]] | 摄取时编译、Raw／Wiki／Schema、运行控制面、Save／Lint／Review 与幻觉回写风险 |

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
| [[wiki/sources/Agent 安全评估：AutoControl Arena 与对齐幻觉]] | 逻辑—叙事解耦、压力与诱惑、对齐幻觉、场景化安全缩放及强弱模型的不同失误机制 |
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
| [[wiki/sources/驾驭工程：Prompt、Context 与 Harness 的边界]] | 怎么问、怎么记、怎么管的职责划分，工作流、权限、代码测试率与模型能力边界 |
| [[wiki/sources/驾驭工程：Harness Engineering 运行系统全景]] | Prompt—Context—Harness 嵌套关系、企业实践、六个模块、旧技术重组与四类风险 |
| [[wiki/sources/驾驭工程：系列完结，下一步该往哪走？]] | Harness 工程系列收束、作者定义与观测、优化、安全三个后续方向 |
| [[wiki/sources/驾驭工程：HarnessX 可进化 Agent Harness]] | Processor 生命周期、AEGIS、确定性闸门、变体隔离与 Cross-Harness GRPO |
| [[wiki/sources/驾驭工程：Claude Code Agent Runtime 架构拆解]] | ReAct 五阶段、七层恢复、四级压缩、投机执行、多 Agent、五层记忆与纵深安全 |

### AI Agent

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/AI Agent 基础：Prompt、Function Calling 与 MCP]] | User Prompt、System Prompt、Agent、工具调用接口与 MCP 服务边界 |
| [[wiki/sources/AI Agent 实践：Pydantic AI 工具调用与消息历史]] | 本地工具注册、`run_sync()`、`all_messages()`、`message_history` 与最小 Harness 边界 |
| [[wiki/sources/AI Agent 框架选型：十大框架与五大范式]] | MCP、A2A、五种架构范式、十大框架地图、四步选型与隐性成本 |
| [[wiki/sources/Agent 记忆：MemRL 运行时强化学习]] | 冻结模型权重，以语义粗筛、Q-value 精排和环境反馈更新实现运行时情景记忆学习 |
| [[wiki/sources/Agent 世界模型：服务于行动的选择性压缩]] | 环境、判断与知识三层结构，检查器和学习型世界模型两条路线，以及记忆过期与验证边界 |
| [[wiki/sources/Agent 安全治理：Claude Fable 5 与 Mythos 5 的分层开放]] | Agentic Execution、百万 Token 上下文、网络安全门控、生命科学双重用途与系统治理 |

### 模型训练与后训练

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/模型训练：梯度下降与均方误差]] | 线性回归、平方误差、参数梯度、学习率、Batch Size 与 MSE |
| [[wiki/sources/模型训练：PyTorch 手写数字识别实战]] | Dataset、DataLoader、CrossEntropyLoss、训练循环、权重保存与 Adam |
| [[wiki/sources/大模型微调：LoRA 低秩适配]] | 预训练与 SFT、全量微调显存、低秩矩阵、Rank 和缩放系数 |
| [[wiki/sources/大模型后训练：强化学习如何选择反馈与算法]] | RLHF、RLAIF、RLVR、DPO、GRPO 的反馈条件，Agent RL 五个工程问题与奖励黑客 |
| [[wiki/sources/大模型后训练：ARLArena 与 SAMPO 稳定 Agentic RL]] | 训练崩塌、序列级裁剪、步骤级优势、动态采样及专项训练比较边界 |
| [[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习]] | 轨迹到技能的蒸馏、冷启动 SFT、技能增强 RL、验证驱动进化及工程限制 |
| [[wiki/sources/大模型后训练：RLM Harness 组合泛化]] | 局部分布内、Context 卸载、程序化子调用及跨长度与跨领域组合泛化 |
| [[wiki/sources/Agent 强化学习基础设施：Kimi K3 AgentENV]] | Partial Rollout、microVM 沙箱、暂停恢复、环境分岔、增量检查点与 Off-Policy 边界 |

### 模型架构

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]] | 原始 Encoder—Decoder 翻译架构、逐 Token 生成、监督与自监督训练，以及 GPT 与 BERT 分支 |
| [[wiki/sources/模型架构：Linear、Activation 与 MLP]] | 线性映射、权重与偏置、梯度下降、ReLU／Sigmoid／tanh／GELU、FFN 与示例 MLP |
| [[wiki/sources/模型架构：PyTorch 手写数字识别]] | Linear 张量形状、MNIST 展平与缩放、三层网络推理、Logits 和 Softmax 维度 |
| [[wiki/sources/模型架构：多头注意力与 QKV]] | Embedding 到 QKV 投影、缩放点积、因果 Mask、Softmax、Value 加权与多头拼接 |
| [[wiki/sources/模型架构：MoBA 混合块注意力]] | 块级稀疏路由、可变长度 FlashAttention、online Softmax 及长上下文效率边界 |
| [[wiki/sources/模型架构：GQA、DSA 与 MSA 长上下文优化]] | GQA 共享 KV、DSA 的 Token 级筛选、MSA 的块级筛选及完整上下文打分成本 |
| [[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性]] | CSA／HCA 两级压缩、mHC、Muon、Anticipatory Routing、OPD 及能力边界 |
| [[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]] | 标准残差的数值膨胀与信息稀释、pseudo-query、Full AttnRes、Block AttnRes 及资源取舍 |
| [[wiki/sources/模型架构：MoE 稀疏专家路由]] | FFN 与 Dense 模型、Router、路由专家、共享专家、激活参数及 DeepSeekMoE 对比边界 |
| [[wiki/sources/模型架构：Engram 参数化记忆查找]] | 参数化 N-gram 查找、多头哈希、上下文门控、CPU 预取及 MoE 记忆分工 |
| [[wiki/sources/长序列建模：Memory Caching]] | 分段隐状态检查点、四种聚合策略及 RNN 长程召回与计算成本的连续权衡 |

### 模型推理优化

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/模型推理优化：DSpark 投机解码]] | 首 Token 容量、半自回归草稿、置信度调度、校准、无损早停及 DeepSeek-V4 线上结果 |
| [[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]] | Prefill 与 Decode、Reasoning Token、KV Cache、Prompt Caching、Batch、任务总成本与 Token FinOps |
| [[wiki/sources/图像重建：DLSS 与 FSR 的时序超分辨率]] | 单帧 Upscaling、亚像素抖动、运动矢量、渲染辅助参数，以及 DLSS／FSR 的 AI 与固定算法路线 |

### 计算基础设施

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/AI 计算硬件：内存带宽、互联与软件生态]] | CPU 与内存带宽、HBM、批量并行、模型与数据并行、NVLink／Infinity Fabric 及 CUDA／ROCm／OpenXLA 生态 |

### 模型原理

| 页面 | 内容 |
| --- | --- |
| [[wiki/sources/大语言模型：Token 与两类 Embedding]] | Tokenizer、Token ID、Token Embedding、RAG Embedding、One-Hot 与对比学习 |
| [[wiki/sources/模型原理：Token Space 与 Latent Space]] | Token 到隐藏表示再回到 Token 的生成路径、分词机制、模型可解释性与 Latent Reasoning |
| [[wiki/sources/大语言模型：思维链的模式匹配与泛化边界]] | DataAlchemy、任务／长度／格式泛化、可见 CoT 与答案不一致及训练分布边界 |
| [[wiki/sources/大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练]] | 1.04T MoE、MuonClip、Data Rephrasing、Agent SFT/RL、原生 INT4 与长程工具调用 |
| [[wiki/sources/大语言模型：Qwen 3.5 的 MoE、混合注意力与应用演示]] | 397B／17B 稀疏 MoE、混合注意力、原生多模态、视觉 Agent 与 Cline API 工作流 |
| [[wiki/sources/多模态推理：ThinkMorph 交错思维链]] | 文本规划与视觉操作交替推进、三种涌现能力、测试时扩展及适用边界 |
| [[wiki/sources/多模态模型：ViT 图像分块与编码]] | 像素输入的规模与语义问题、图像分块、局部特征、Transformer 编码器和多模态输入链路 |
| [[wiki/sources/多模态模型：架构、数据、推理与检索]] | 视觉编码、模态接口、数据工程、MCoT、跨模态检索与多模态 RAG 的完整链路 |
| [[wiki/sources/多模态推理：视觉原语与 Reference Gap]] | Perception Gap 与 Reference Gap、框和点作为推理变量、类 LLaVA 架构及 7056× 工程压缩链路 |
| [[wiki/sources/多模态推理：视觉原语的数据、训练与奖励]] | 两阶段数据过滤、框点专家训练、Unified RFT、OPD、三层奖励及拓扑评测边界 |

## 跨资料综合

| 页面 | 内容 |
| --- | --- |
| [[wiki/syntheses/大模型后训练：从模仿到行为选择]] | 从 SFT 模仿到偏好与强化学习的路线选择、轨迹训练、技能蒸馏和独立评测 |
| [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]] | 离散 Token、内部连续表示与显式多模态交错思维的职责、成本和审计边界 |
| [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]] | Harness 的信息、行动、控制、安全、观测与训练职责，以及静态外壳到可进化对象的边界 |
| [[wiki/syntheses/AI Agent：从工具调用到可信行动]] | 工具调用、四类状态、世界模型、长链可靠性与确定性放行 |
| [[wiki/syntheses/长上下文模型架构：共享、筛选、压缩与可增长记忆]] | GQA、稀疏筛选、CSA／HCA 与 Memory Caching 的作用层和成本边界 |
| [[wiki/syntheses/深层模型训练稳定性：残差、更新与路由]] | 梯度、残差、矩阵正交化、注意力幅度、MoE 路由与策略比率失稳 |
| [[wiki/syntheses/多模态推理闭环：感知、指代、操作与验证]] | ViT、多模态接口、视觉原语、交错思维链与程序化奖励 |

## 当前来源边界

- 资料中的模型表现、价格、延迟、Token、准确率和硬件数字保留来源属性，不能脱离实验或案例条件外推。
- AI 计算硬件资料中的 Llama 3.1 405B、EPYC 9965、DDR5、HBM、H100、B200／B300、MI355X、NVLink 与 Infinity Fabric 数字采用视频发布时的配置或教学估算，没有提供统一精度、批量、功耗、延迟和软件版本下的实测；开场“MI350 约 15,000 美元”与后文画面“MI355X 25,000 美元以上”不是同一精确型号和价格口径。
- Claude、Codex、MCP 和其他工具的命令及 API 行为具有版本时效性，实际使用前需核对对应官方文档。
- Claude Code Runtime 资料来自作者对其所称“网上流出的部分核心源码”的审计，但视频与简介没有提供源码仓库、Commit、版本号或可复核快照；1,884 个 TypeScript 文件、42 种以上工具及内部机制只保留为该资料对 2026 年 4 月 2 日所分析材料的描述，不能外推为其他版本的固定事实。
- AI Agent 基础资料关于 Function Calling 厂商差异与开源模型支持情况的判断对应 2025 年 5 月；模型负责提出工具调用，Agent 或应用负责执行工具这一职责边界不随具体接口名称改变。
- Prompt 到上下文治理资料关于网页产品内置推理、消息角色和 Agent 实现的描述对应 2025 年 8 月；实际应用需要按当前 API、模型和状态管理方式重新核对。
- 评估工程系列共八期，当前知识库已经完整收录；第八期以合同驱动架构收束从代码评分到企业治理的演进。
- 驾驭工程收尾篇采用作者的宽泛 Harness 定义；其术语范围与本库此前“Harness 作为 Prompt、Context 与 Loop 共同外壳”的来源表述并存，不合并为统一定义。
- Prompt、Context 与 Harness 对照资料采用“怎么问、怎么记、怎么管”的窄口径；作者关于未发布模型直接超过其 AI 系统的经历没有公开模型、任务、评测方式或具体差距，只能支持持续比较工程收益与基础模型升级，不能证明 Harness 必然失效。
- Harness Engineering 全景资料中的 OpenAI、Anthropic、Google DeepMind、Vercel、Stripe 与 Manus 案例对应不同组织、系统和评测条件；百万行代码、移除 80% 工具、Aletheia 成绩及 BrowseComp 的 0.24%／0.87% 不能合并为统一效果证明。资料采用 Harness 包含 Context、Context 包含 Prompt 的运行系统口径，与库内其他来源口径并存。
- Agent 框架资料反映 2026 年 3 月的版本、生态与商业模式；实际选型前需要重新核对官方文档。视频内 Dify Star 数存在旁白与画面差异，Agno 也未获得与其余框架同等篇幅的分析。
- 当前大模型后训练综合由 LoRA、强化学习路线总览、ARLArena、Kimi K2 Thinking、SKILLRL、RLM Harness、HarnessX、MemRL 与 Kimi K3 AgentENV 共同支撑；其中数字来自各自模型、任务、论文和基础设施设置，不构成统一数据集上的效果排名或其他 Agent 的收益保证。ARLArena 中 GPT-5.2、o3 与 Qwen3-4B SAMPO 没有接受相同的环境专项训练，不能据此形成通用模型能力排名。
- LoRA 资料中的 26GB 全量微调与 8GB LoRA 显存账单来自 `DeepSeek-R1-Distill-Qwen-1.5B`、FP16 参数、FP32 AdamW 状态、$r=8$ 和资料采用的简化假设；不能外推为其他模型、精度、序列长度、批量或训练框架的固定结果。资料对 LoRA 有效性的解释属于侧面理解，不是严格证明。
- Memory Caching 的实验数字来自资料转述的特定模型、规模和任务；其模型架构层记忆机制不等同于应用层上下文治理。
- MoBA 的复杂度、百万 Token 召回及约 6.5 倍计算时间改进来自资料转述的 Llama-8B-1M、块划分、Top-k 与对应硬件设置；不能外推为其他模型和部署的保证。
- GQA 资料中的 1/16 KV 缓存来自 64 个 Query Head、4 个 KV 组的教学配置；打分注意力比主注意力小几十倍也是来源概括，不能外推为其他模型的固定资源比例。DSA 与 MSA 的筛选效果仍受 Top-N、分块、训练和任务长程依赖影响。
- 多头注意力资料中的“语法表”“需求表”“内容矩阵”和可读 Attention Head 均为教学类比，不代表真实隐向量维度可以逐项命名；1600、7168 与商业模型上万维的表述保留来源对应模型和推测边界。
- Linear 与 MLP 资料中的 GPT-2 XL `1600→50257`、示例 MLP `1→128→256→1`、33537 个参数和 2000 条数据来自对应模型与教学实验；“从0开始一起学大模型”合集顺序不等于视频正式标注的期数。
- 梯度下降资料中的 $y=1.5x+2.5$、1000 条数据、Batch Size 100、Learning Rate 0.01、1000 次训练及 $y=1.4994x+2.4845$ 均属于线性回归教学实验；批量和超参数不构成其他模型的默认配置或收敛保证。
- PyTorch 训练实战中的 Batch Size 50、Epoch 5、Learning Rate 0.01、`784→256→128→10` 模型和训练集第 90 条样本结果属于对应 MNIST 教学实验；训练集样本预测不能替代独立测试集上的泛化评估。
- Attention Residuals 资料只讲解机制与开销，没有介绍论文实验成绩；人物识别层与关系层属于教学类比，不能据此认定真实模型各层具有可直接命名的固定分工。
- MoE 资料中的 256 个路由专家与每次选择 8 个专家属于 DeepSeek-V3 配置；144.6B／22.2B、67.4B、585.6T／2057.5T 和 Pile Loss 来自 DeepSeekMoE 对比设置，不能直接外推为其他模型的速度或 Token 价格。
- Engram 的 26.7B／3.8B 等预算对照、5.7B 记忆参数、75%～80% MoE 配比、100B CPU 内存查表和 2.8% 吞吐下降均来自资料转述的对应模型、任务、H800 与预取流水线；“20% 记忆、80% 动态计算”是作者对实验曲线的解释，不构成其他领域的固定比例。
- Tokenization、SuperBPE、T-Free、SAE、Coconut 和 Soft Thinking 的性能或节省数字来自资料转述的对应研究设置；不能据此外推到其他模型、语言和任务。Latent 表示也不构成模型具有意识或主观体验的证据。
- Kimi K2 Thinking 的架构参数、15.5T Token 稳定训练、INT4 约 2 倍生成速度、200～300 次连续工具调用和评测数字来自资料对应的模型、硬件、工具与预算设置；不能外推为其他部署的收益或长程可靠性保证。
- Qwen 3.5 资料中的 397B／17B、512 个专家、75%／25% 混合注意力、20,000 个并行环境、60% 成本下降、8 倍吞吐、长上下文最高 19 倍和 201 种语言均采用视频口径；视频没有提供完整基准协议。100 万 Token 免费额度、640GB 本地显卡需求和小模型开放状态对应 2026 年 2 月 20 日，实际使用前需要重新核对。
- Claude Fable 5 与 Mythos 5 资料中的上下文、价格、漏洞利用、越狱、蛋白设计和大代码库迁移数字来自 2026 年 6 月的厂商材料与对应评测，不能外推为其他环境的固定能力或安全保证。官方视频标题写有“被美国政府紧急叫停”，但正文、简介和画面没有提供可核实的机构、命令、日期或措施，本库不把该说法作为事实结论。
- DeepSeek V4 资料中的 1.6T／49B、33T 训练 Token、百万 Token 上下文、27% 单 Token FLOPs、10% KV Cache、4.6GB／15GB／31GB 对照、0.00089 美元估算及各项评测均对应视频和技术报告采用的模型、精度、模式、预算与 Serving 口径；不能外推为其他部署的固定成本或所有任务上的能力排名。
- ThinkMorph 的实验数字来自 BAGEL-7B、24,990 条训练轨迹及对应基准；模式切换的 5.3% 在讲解文字与论文图注中分别归于 MMVP 和 Chart Refocus，本库保留该来源冲突。
- 多模态技术地图中的架构提升、训练数据、推理成绩、检索指标和延迟来自资料转述的不同论文设置；不能合并为统一排行榜或外推为其他模型、任务和部署的普遍规律。
- 视觉原语资料的 7,056× 是原始像素数与视觉 KV Cache 条目数之间的工程比值；90 个缓存条目与 77.2 平均分来自论文指定分辨率和七项选定评测，不代表模型整体能力。
- 视觉原语下集的 4,000 万样本、66.9% 迷宫导航和 56.7% 路径追踪结果来自报告的特定数据、低推理预算与评测协议；报告没有标准消融表，不能分别量化视觉原语、框点分训、OPD 和 CSA 的贡献。
- RLM Harness 的组合泛化结果只验证了一个 30B 底座和适合切块的任务；训练样本耗时为直接训练的 1.5～3.0 倍，不能外推为所有模型或高度耦合任务的通用结论。
- DSpark 的接受长度、草稿接受率和 60%～85% 单用户生成速度提升来自资料转述的对应实验及 DeepSeek-V4 线上负载；不能外推到其他模型、硬件、批量或流量结构。
- Token API 资料中的价差、输入输出倍率、长上下文分档、缓存节省上限和 Batch 折扣反映视频发布时的平台规则概括；实际采购需核对具体模型、区域、服务等级和当前官方价格页。
- MemRL 的 3.8 个百分点平均提升、探索密集型任务 6.2 个百分点提升与约零额外推理成本来自资料转述的四项实验；不能外推为其他 Agent 的效果或成本保证。
- RAG 个人知识库资料中的 1536 维与 3072 维分别对应 `text-embedding-3-small` 和 `text-embedding-3-large`；Pinecone、ChromaDB 及 PostgreSQL 配合 pgvector 只是来源列举的存储选择，不构成固定架构要求。
- LLM Wiki 资料中的八至九类页面、单来源触达 8～15 页、八步 Ingest、Thesis Mode 的 5～10 个 Agent、约 100 个来源规模与运行控制文件均采用视频对 2026 年 4 月公开实现的归纳，不构成统一规范。GraphRAG-Bench、Token 与延迟结论也只能用于资料所述条件。
- DRAG 与 IterDRAG 的平均准确率、CoT 对比和参数热图来自资料转述的 Gemini 1.5 Flash、四项基准及对应配置空间；不能外推为其他 RAG 系统的固定参数或收益保证。
- Kimi K3 AgentENV 的沙箱、镜像、最低检查点与恢复延迟、98% 等待占比和最高 6.5 倍内存超配来自对应训练与评估工作负载；不能外推为其他集群的容量、延迟或资源利用保证。
- Agent 世界模型资料中的 78%、90%、约 4%、71% 和 94% 来自对应下棋与易犯规游戏设置；合成世界训练略微超过真实环境训练也只属于 Qwen-AgentWorld 的对应实验，不能外推为所有 Agent 或训练环境的收益。
