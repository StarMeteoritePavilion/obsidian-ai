---
title: DeepSeek V4 硬核全面拆解｜1.6万亿参数+百万级上下文+33万亿token同时不崩，CSA、mHC、Muon、OPD四大核心模块逐条拆解
source: https://www.bilibili.com/video/BV153oRBXEsG
author: 唐国梁Tommy
created: 2026-04-25
tags:
  - AI
  - 模型原理
  - 模型架构
  - DeepSeek-V4
  - 长上下文
  - MoE
---

> DeepSeek V4 模型架构独立专题

DeepSeek-V4 Preview 技术报告不是围绕单一算法展开，而是回答一个系统工程问题：怎样把 Transformer 与 MoE 架构同时推到百万级上下文、1.6 万亿参数和 33 万亿训练 Token，并保持训练与推理可用。

DeepSeek-V4-Pro 在百万 Token 上下文下的单 Token 推理计算量约为 DeepSeek-V3.2 的 27%，KV Cache 约为后者的 10%；Codeforces 评分达到 3206，Putnam-2025 形式化证明获得 120 分满分。这些结果来自对应技术报告和评测设置，不能脱离模型模式、推理预算与测试协议理解为所有任务上的全面领先。

## 长上下文首先受显存带宽限制

推理模型通过 Test-Time Scaling 增加思考长度，但更长的思考也意味着更长的上下文。标准 Attention 的计算会随序列长度快速增长；KV Cache 虽然避免重复计算历史 Key 和 Value，却要求每生成一个新 Token 都从显存读取已有 KV。

百万 Token 场景下，每层生成一个新 Token 需要读取接近 1GB 的 KV；61 层累计约为 60GB。按 H100 级别的 HBM 带宽估算，仅读取 KV 就需要约 20ms。瓶颈因此不只是“算不过来”，更是“读不过来”：GPU 算力增长速度快于 HBM 带宽，长上下文逐渐成为硬件数据搬运问题。

已有路线分别存在边界。MLA、GQA 主要压缩 Head 维度，序列维度仍会随上下文增长；NSA、DSA 等稀疏注意力仍需让 Indexer 对完整 KV 打分；Mamba、RWKV 等线性架构使用固定状态，但长程精确检索存在取舍。DeepSeek V4 的选择不是押注单一路线，而是把压缩与稀疏作为两条正交轴组合起来。

## CSA 与 HCA：先压缩，再选择

Compressed Sparse Attention（CSA）分两步处理长序列。第一步沿时间维度把相邻 Token 按学习到的权重合并，以四倍压缩率把百万长度缩减为约 25 万个压缩 KV。第二步针对当前 Query，只从压缩后的 KV 中选择最相关的 Top-k 项执行 Attention；DeepSeek-V4-Pro 的配置为 1024 项。

DeepSeek-V3.2 的 DSA 直接从原始百万序列中选择 2048 个位置。V4 先压缩四倍，再从压缩结果中选择 1024 项，Indexer 的打分对象降至原来的四分之一。相邻压缩块采用重叠窗口，每个压缩块覆盖约八个原始 Token 的语义，因此 1024 个压缩块等效覆盖约 8000 个原始 Token。压缩减少打分范围，稀疏选择控制主 Attention 的实际输入，两项收益能够叠加。

Heavily Compressed Attention（HCA）承担互补角色。它以 128 倍压缩率把百万 Token 压到约 8000 个 KV，并在这个尺度上直接执行 Dense Attention，不再做 Top-k 选择。CSA 与 HCA 在层间交错：CSA 保留较细粒度的局部选择，HCA 提供经过重压缩的全局视野。作者把这种组合类比为 FPN 特征金字塔中的多尺度感受野。

资料给出的百万 Token KV Cache 估算中，DeepSeek-V4-Pro 约占 4.6GB，DeepSeek-V3.2 的 MLA 约为 15GB，传统 GQA-8 基线约为 31GB；按报告完整 Serving 口径，V4-Pro 的 KV Cache 约为 V3.2 的 10%。具体数值依赖模型配置、精度和统计口径。

## mHC：约束多通道残差流

传统 Transformer 只有一条残差流，跨层信息都挤在同一个隐藏向量中。Hyper-Connections（HC）把残差流扩展为多通道；DeepSeek V4 使用四个通道，以增加跨层表示能力。未经约束的多通道混合却可能逐层放大信号：每层即使只放大少量，深层叠加后也会同时放大前向信号和反向梯度，造成 Loss Spike。

Manifold-Constrained Hyper-Connections（mHC）把残差混合矩阵约束为双随机矩阵：每一行之和为 1，每一列之和也为 1，所有元素非负。这类矩阵位于 Birkhoff 多面体中，其谱范数不超过 1，连续相乘后仍保持同类约束，因此残差变换是非膨胀的。

矩阵投影通过 Sinkhorn-Knopp 算法完成：先保证矩阵元素为正，再交替执行行归一化与列归一化，约迭代 20 轮。这个约束允许信号在残差通道间重新分配，却限制整体强度持续放大，为深层多通道残差流提供稳定性边界。

## Muon：从逐元素归一化转向矩阵正交化

DeepSeek V4 将大部分二维权重矩阵的优化器从 AdamW 改为 Muon；Embedding、预测头、mHC 的静态偏置与门控参数，以及 RMSNorm 权重仍由 AdamW 更新。

AdamW 按元素处理梯度，无法直接利用二维权重矩阵的整体结构。Muon 则对矩阵更新进行近似正交化：通过 Newton-Schulz 迭代把不同方向的奇异值拉向 1，弱化各方向拉伸尺度的差异，保留主要方向信息。

V4 使用 Hybrid Newton-Schulz，把十次迭代分为两段：前八步采用更激进的系数，使奇异值快速接近 1；后两步改用更保守的系数精修。关键正交化和梯度同步可以在 BF16 下完成，资料称通信量因此减半。Muon 对权重更新的约束与 mHC 对残差流的约束共同形成非膨胀路径，分别控制参数更新与跨层信号传播。

## Anticipatory Routing：打断 MoE 的即时正反馈

MoE 训练可能在路由概率与专家参数之间形成正反馈。某一步若把一批特征范数异常大的 Token 分给同一专家，该专家的梯度会向这批 Token 偏移；下一步的相似 Token 随后更容易再次被路由到该专家，最终触发 Loss Spike。在 33 万亿 Token 的训练规模下，单次异常就可能破坏长时间训练。

Anticipatory Routing 把路由索引与当前骨干网络参数解耦：当前步骤的特征、Gating 权重和专家前向使用当前参数，路由索引则由若干步之前的路由参数预先计算并缓存。时间延迟打断了路由决策与专家更新之间的即时闭环。系统只在检测到 Loss Spike 时短暂回滚并启用该机制，稳定后再恢复标准训练。

DeepSeek V4 还使用 SwiGLU Clamping 抑制异常值：线性分支限制在 $[-10,10]$，门控分支的上界限制为 10。技术报告明确表示，这两项方法有效稳定了训练，但其完整理论机制仍有待解释。

## OPD：先分别训练专家，再统一蒸馏

DeepSeek-V3.2 的 Mixed RL 把数学、代码、对话等多个 Reward Pipeline 放在同一阶段优化。不同领域的奖励可能相互稀释甚至对冲，奖励调度的组合成本也会随领域增加。

DeepSeek V4 改用 On-Policy Distillation（OPD）。系统先从同一 Base 模型出发，分别训练数学、代码、Agent 和指令遵循等领域专家；Student 再沿自己的当前策略生成轨迹，并在每个 Token 位置与对应 Teacher 的完整词表概率分布对齐，优化 Reverse KL。

作者用两组对比解释这个选择。Forward KL 倾向覆盖 Teacher 的多种可能路径，容易形成 Mode Averaging；Reverse KL 更惩罚 Student 进入 Teacher 低概率区域，使其靠近某个高概率 Mode，形成 Mode Seeking。轨迹由 Student 自己生成，而不是直接复制 Teacher 轨迹，因此训练发生在 Student 实际会访问的状态上，缓解固定示范带来的 Covariate Shift。

不同领域的 Teacher 可以独立训练并加入 Teacher 池，不必重做统一的奖励调度。资料称，这使增加新领域的迭代周期从数周缩短到数天。

## 五条优化轴的共同性质

DeepSeek V4 的技术组合可以概括为五条轴：

1. CSA 进行四倍轻压缩，HCA 进行 128 倍重压缩。
2. 稀疏选择发生在压缩后的序列上，而不是原始长序列上。
3. mHC 把残差混合矩阵约束到 Birkhoff 多面体。
4. Muon 通过 Hybrid Newton-Schulz 约束二维权重矩阵的更新。
5. OPD 以 Reverse KL 把多个领域 Teacher 的能力合并到统一 Student 中。

这些模块分别处理序列长度、稀疏计算、残差信号、参数更新、MoE 路由和后训练整合。作者把它们的共同设计哲学概括为“约束优于修补”：通过非膨胀映射或时间解耦，让信号留在稳定范围内，而不是在训练崩溃后依赖临时超参数补救。

## 成绩、短板与复现边界

DeepSeek-V4-Pro-Max 的 Codeforces 评分为 3206；Putnam-2025 在混合形式化—非形式化推理与较大计算预算下达到 120/120。百万 Token 上下文下，资料估算每生成一个 Token 的成本约为 0.00089 美元。这些指标来自不同任务、模式和计算预算，不能合并成统一能力排名。

长上下文仍有精细检索损失。MRCR 1M 中，DeepSeek-V4-Pro 为 83.5，Claude Opus 4.6 为 92.9，相差 9.4 分；作者把这项差距解释为 CSA 压缩损失的显式表现。因此，法律合同逐条比对、代码变量重命名等需要精确定位单个 Token 的任务未必是它的优势场景。

Agent 评测也没有全面领先闭源模型。Terminal Bench 2.0、BrowseComp、HLE w/ tools 与 GDPval-AA 的结果分布不同，不能把它们概括成统一的固定差距。作者认为可能的原因包括闭源模型拥有实时 Agent RL 环境，而 V4 使用重放静态轨迹；OPD 的 Teacher 池也可能缺少足够强的 Agent Teacher。这些属于作者对结果的解释，不是报告已经分别验证的因果结论。

开放数学泛化同样不均衡：HMMT 2026 Feb 上 DeepSeek-V4-Pro-Max 为 95.2，Apex 上为 38.3，后者低于 Gemini-3.1-Pro 的 60.9。训练内相近题型的高分不能直接证明面对新型题目时也有同等表现。

复现难度来自模块间的深度耦合与配套系统。模型使用 DeepGEMM、TileLang、3FS、DeepSeek Elastic Compute（DSec）和 DualPipe 等基础设施；mHC、Muon、CSA 的归一化与 Attention 数值边界也相互依赖。任何单点简化都可能改变其他模块的稳定性前提。

DeepSeek V4 证明了数学约束、稀疏计算和系统基础设施可以共同把百万上下文推向可用范围，但也留下一个开放问题：未来模型会继续走向深度耦合与显式数学约束，还是回到更易替换的模块化架构。当前结果说明这条路线能够工作，并没有消除精细检索、Agent 泛化、开放数学与工程复现的代价。
