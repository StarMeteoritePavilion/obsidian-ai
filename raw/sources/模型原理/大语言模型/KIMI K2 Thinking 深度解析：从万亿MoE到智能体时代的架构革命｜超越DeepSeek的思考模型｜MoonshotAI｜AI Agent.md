---
title: KIMI K2 Thinking 深度解析：从万亿MoE到智能体时代的架构革命｜超越DeepSeek的思考模型｜MoonshotAI｜AI Agent
source: https://www.bilibili.com/video/BV1sJCnBGESj
author: 唐国梁Tommy
created: 2025-11-12
tags:
  - AI
  - 模型原理
  - 大语言模型
  - Kimi-K2-Thinking
  - MoE
  - Agent
---

> Kimi K2 Thinking 独立专题

Moonshot AI 推出的 Kimi K2 系列以万亿参数级 Mixture-of-Experts（MoE）架构为基础。Kimi K2 Base 提供基础模型能力，Kimi K2 Thinking 则通过后训练把函数调用变成推理中的原生动作，使模型按照“思考—行动—再思考”的方式与工具和环境交互。

这套路线同时处理六类问题：MuonClip 保持万亿参数训练稳定；Data Rephrasing 提高高质量数据的 Token 效率；大规模 Agent 数据合成提供工具使用轨迹；原生 INT4 降低部署成本；长程训练支持 200～300 次连续工具调用；Test-Time Scaling 在推理阶段增加思考与工具调用预算。

资料展示了三个应用案例：构建复杂前端编辑窗口、动态解释梯度下降，以及模拟病毒攻击血液细胞的生物过程。这些案例用于说明模型不仅生成文本，也能组合代码、交互界面和任务规划。

## 1.04T MoE 架构

资料给出的 Kimi K2 架构包含 1.04T 总参数，每个 Token 激活 32B 参数。模型共有 384 个专家，每次由路由器选择 8 个专家，并加入 1 个共享专家共同处理。第一个 Transformer Block 使用标准密集 FFN，从第二个 Block 开始引入 MoE。

更早使用 MoE、增加专家数量并保持较低激活参数量，使模型容量和单次推理成本分离。更多、小型的专家用于形成更细的知识分工，但每个 Token 只调用其中一小部分。

词表规模为 160K，资料将其作用概括为提高多语言和专业术语的编码效率。Kimi K2 Thinking 支持 256K 上下文。注意力机制采用 Multi-head Latent Attention（MLA），通过潜在表示压缩 Key-Value 状态，减少 KV Cache 占用。

## 与 DeepSeek R1 的架构取舍

资料将 Kimi K2 Thinking 与 DeepSeek R1 作了结构对比：

| 配置 | Kimi K2 Thinking | DeepSeek R1 |
| --- | ---: | ---: |
| 总参数量 | 1T | 671B |
| 词表规模 | 160K | 129K |
| 注意力头 | 64 | 128 |
| MoE 专家 | 384 | 256 |
| 激活参数量 | 32B | 37B |
| 非 MoE 层 | 第 1 个 Block | 前 3 个 Block |

这组设计减少注意力头、增加专家数量、更早引入 MoE，并把单 Token 激活参数控制在 32B。资料据此将 Kimi K2 的取舍概括为：用更大的总容量追求性能上限，同时降低每次推理实际动用的参数量。表中数字对应资料采用的模型版本，不能外推为所有 Kimi 与 DeepSeek 版本的固定差异。

## MuonClip：兼顾 Token 效率与训练稳定性

万亿参数预训练面对两个问题：有限高质量数据应产生更高的有效学习信号，大规模训练还必须避免数值不稳定。Kimi K2 使用 Muon 优化器提高 Token 效率，但 Muon 扩展到大模型时可能引发 Attention Logit 爆炸。Logit 在 Softmax 之前由 Query 与 Key 的点积产生；数值过大时，Softmax 会接近 One-hot 分布，梯度可能爆炸或消失，最终造成 Loss Spike 甚至训练发散。

两种常见方案各有局限。Logit Soft-Cap 直接限制最终 Logit，但 Query 与 Key 的点积在限制前已经变大；QK-Norm 通过归一化 Query 和 Key 控制点积，却难以直接适配 MLA 中没有完整物化的 Key 表示。

### QK-Clip 从权重端限制 Logit

QK-Clip 不直接修改 Logit，而是在优化器完成权重更新后，根据各注意力头当前 Batch 的最大 Logit 判断是否干预。Kimi K2 使用阈值 $\tau=100$：未超过阈值时权重保持不变；超过阈值时，仅按头缩放产生 Logit 的 Query 与 Key 投影权重。

这种机制不改变当前训练步骤的前向和反向传播，只把已经观测到的最大 Logit 作为事后监控信号。按头干预而不是缩放整层，可以减少对正常注意力头的影响。适配 MLA 时，非共享组件按相应比例缩放，共享组件保持不变，避免一个注意力头的异常影响其他头。

MuonClip 把原始 Muon、Weight Decay、Consistent RMS Matching 和 QK-Clip 组合为一个优化器。资料引用的训练曲线显示，普通 Muon 的最大 Attention Logit 会超过 1,000；MuonClip 在达到 100 后执行限制，随后 Logit 回落到稳定范围。Kimi K2 据此完成 15.5T Token 的预训练，资料称全程没有 Loss Spike。

## Data Rephrasing：改写比机械重复更有效

高质量人类数据日益稀缺，机械重复同一数据会增加过拟合风险。Data Rephrasing 在保持事实内容的前提下，生成不同风格和视角的表达，提高每个 Token 提供的有效学习信号。

数据改写分为三步：

1. 通过 Prompt Engineering 引导大模型从多种风格和视角忠实改写原文。
2. 将长文档分块，自回归地逐块改写，再重新拼接以保持全局连贯。
3. 比较改写段落与原文的语义一致性，执行 Fidelity Verification。

资料引用的早期 K2 Checkpoint 在 SimpleQA 上得到以下结果：原始 Wiki 文本训练 10 个 Epoch 的准确率为 23.76；改写 1 次并训练 10 个 Epoch 为 27.39；改写 10 次、每份训练 1 个 Epoch 为 28.94。该实验说明，在对应数据与早期 Checkpoint 上，多样化改写比机械重复提供了更高的准确率，不代表任意数据经过更多改写都会持续提升。

## Kimi K2 Thinking 的 Agent SFT

Kimi K2 Thinking 的后训练首先通过 SFT 教模型使用工具。真实世界的工具调用轨迹成本高、隐私约束多，也难以大规模获取，因此训练使用三阶段 Agent 数据合成系统。

### 工具规范生成

工具库包含 3,000 多个真实世界工具规范，以及 20,000 多个合成工具规范。真实工具来自 GitHub、MCP 等来源，合成工具用于扩展不同专业领域和使用场景。

### Agent 与任务生成

系统从工具库中抽取工具组合，生成数千个具有不同能力、领域和行为模式的 Agent，再为它们设计从简单到复杂的任务。每项任务都带有明确的成功标准 Rubric，为后续筛选提供依据。

### 轨迹生成与过滤

轨迹生成系统包含三个组件：User Agent 模拟不同沟通风格的用户；Tool Simulator 执行工具调用并返回反馈；Judge Agent 根据预设 Rubric 评估交互轨迹，只保留成功轨迹用于训练。

模拟环境便于扩展，但真实性有限。编码和软件工程等领域因此采用 Hybrid Approach，把模拟工具与真实执行沙箱结合，让模型接触真实执行结果和失败反馈。

## 强化学习的两类奖励

SFT 之后的强化学习用于提高 Token 效率和泛化能力，特别是主观偏好任务与复杂推理任务。奖励分为两类。

第一类是具有明确对错标准的 Verifiable Rewards“Gym”（RLVR），覆盖数学与 STEM、编码与软件工程，以及 Faithfulness。数学可核对答案，代码可运行测试，忠实性可用专门模型判断回答是否得到来源支持。

第二类是 Self-Critique Rubric Reward，适用于创意写作等没有唯一答案的主观任务。K2 同时扮演 Actor 和 Critic：Actor 生成多个回答，Critic 按清晰度、客观性和避免 Sycophancy 等 Rubric 比较、排序并产生偏好信号。Critic 又持续使用 Verifiable Rewards“Gym”的客观信号校准，使主观判断建立在部分可验证反馈上。

强化学习还采用三项改进：Budget Control 惩罚过长回答，鼓励简洁生成；PTX Loss 在强化学习目标中加入预训练损失，降低灾难性遗忘；Temperature Decay 让训练逐步从探索转向利用并促进收敛。

经过 SFT 与 RL，函数调用成为模型推理流程中的原生动作。训练对象不只是最终答案，还包括怎样规划、调用工具、读取环境反馈并继续思考。

## 原生 INT4 与推理架构

Kimi K2 Thinking 在后训练阶段采用 Quantization-Aware Training（QAT），对 MoE 组件进行 Weight-only INT4 量化。资料称原生 INT4 使生成速度提高约 2 倍，显著降低 GPU 显存占用，同时性能接近无损；引用的公开评测结果也以 INT4 精度运行。该收益取决于资料对应的模型、硬件和推理实现，不能直接外推到其他部署。

架构侧继续通过两个选择降低推理成本：注意力头由对比模型的 128 个减少到 64 个，从而降低长上下文中的注意力计算和 KV Cache；MoE 从第二个 Block 开始，使更多计算进入稀疏专家路径。

## Test-Time Scaling 与长程工具调用

Kimi K2 Thinking 不只在训练时扩大参数和数据，也在推理阶段增加思考时间与工具调用预算。资料将这种 Test-Time Scaling 描述为：随着允许的推理时间和调用次数增加，复杂任务表现可以继续提高。

长程 Agent 任务的关键是维持目标和上下文一致性。资料称，一些对比模型在 30～50 步工具调用后开始性能下降或偏离目标，而 Kimi K2 Thinking 可以完成 200～300 次连续工具调用。它由此能够执行网页浏览、知识检索、编程等多步骤组合任务。这一范围来自对应演示和评测设置，不是任意任务上的可靠性保证。

## 评测结果及其边界

资料从推理、Agent Search 和编码三类任务比较 Kimi K2 Thinking、GPT-5 与 Claude Sonnet 4.5。

推理任务中，HLE 使用工具时，Kimi K2 Thinking 为 44.9，GPT-5 为 41.7，Claude Sonnet 4.5 为 32.0；Heavy 设置下，Kimi K2 Thinking 为 51.0，GPT-5 为 42.0。AIME 2025 的 Heavy 设置中，Kimi K2 Thinking 与 GPT-5 都为 100.0。

Agent Search 中，BrowseComp 分别为 60.2、54.9 和 24.1，Seal-0 分别为 56.3、51.4 和 53.4；但 BrowseComp-ZH 上 Kimi K2 Thinking 为 62.3，略低于 GPT-5 的 63.0，高于 Claude Sonnet 4.5 的 42.4。因此，不能把 Agent Search 结果概括为 Kimi K2 Thinking 在每一项都领先。

编码任务中，Kimi K2 Thinking 在 SWE-bench Verified 为 71.3，低于 GPT-5 的 74.9 和 Claude Sonnet 4.5 的 77.2；SWE-bench Multilingual 为 61.1，高于资料所列 GPT-5 的 55.3；SciCode 为 44.8，略高于 GPT-5 的 42.9 和 Claude Sonnet 4.5 的 44.7。

这些数字来自资料引用的具体工具、推理预算和评测协议，只能用于理解对应设置下的能力分布，不能代表模型在所有任务中的普遍排名。

## API 演示留下的身份边界

作者通过后台配置的 Kimi K2 Thinking API 询问模型身份。模型回答自己属于 Kimi 系列，却没有进一步准确说出 Kimi K2 Thinking。这个简单案例不构成能力评测，但说明模型的自我报告不能代替调用端保存的模型标识和请求配置。

## 结论

Kimi K2 Thinking 的技术链不是单一架构创新，而是把大容量 MoE、稳定预训练、数据改写、Agent 轨迹合成、强化学习、量化部署和测试时扩展组合起来。MoE 分离总容量与激活成本，MuonClip 支撑 15.5T Token 稳定训练，Data Rephrasing 提高高质量数据的利用率，SFT 与 RL 则把工具调用训练成推理中的原生行为。

这种组合使模型更接近可执行长程任务的 Agent 核心，但 200～300 步调用能力、INT4 收益和评测优势都依赖具体环境、工具、预算与验证协议。真正的工程价值不只是模型能够长时间思考，而是每次行动都有明确工具接口、真实环境反馈、成功标准和独立检查。
