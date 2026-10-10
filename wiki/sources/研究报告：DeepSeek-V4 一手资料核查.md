---
title: 研究报告：DeepSeek-V4 一手资料核查
source: https://arxiv.org/abs/2606.19348
author: opencode research agent
published: 2026-04-26
ingested: 2026-10-10
updated: 2026-10-10
tags:
  - AI
  - DeepSeek-V4
  - 一手核查
  - 模型架构
  - 调研
---

# 研究报告：DeepSeek-V4 一手资料核查

本文对应 [DeepSeek-V4 技术报告 v1](https://arxiv.org/html/2606.19348v1) 中的 Preview 模型。模型架构、训练方法和成绩以该版本论文为依据；代码分析使用固定提交，缓存复算与实际服务显存分别说明。

相关页面：[[wiki/sources/模型架构：DeepSeek V4 的长上下文与训练稳定性]]、[[wiki/syntheses/长上下文模型架构：共享、筛选、压缩与可增长记忆]]；视频讲解见 [[raw/sources/模型原理/模型专题/DeepSeek V4 硬核全面拆解｜1.6万亿参数+百万级上下文+33万亿token同时不崩，CSA、mHC、Muon、OPD四大核心模块逐条拆解]]。

## 模型规模与版本

| 配置 | DeepSeek-V4-Pro | DeepSeek-V4-Flash |
| --- | --- | --- |
| 总参数 / 每 Token 激活参数 | 1.6T / 49B | 284B / 13B |
| 预训练 Token | 33T | 32T |
| 骨干层数 / 隐藏维度 | 61 / 7168 | 43 / 4096 |
| 共享 / 路由 / 每 Token 激活路由专家 | 1 / 384 / 6 | 1 / 256 / 6 |
| 专家中间维度 | 3072 | 2048 |
| CSA 压缩比 / Top-k | 4 / 1024 | 4 / 512 |
| HCA 压缩比 | 128 | 128 |
| 注意力头数 / 头维度 | 128 / 512 | 64 / 512 |
| 局部滑动窗口 | 128 Token | 128 Token |
| 原生上下文 | 1M Token | 1M Token |

Pro 前两层为 HCA，此后交错使用 CSA 与 HCA；Flash 前两层为纯滑动窗口注意力，此后交错使用 CSA 与 HCA。两者均使用四通道 mHC、20 次 Sinkhorn-Knopp 迭代及深度为 1 的多 Token 预测模块。以上数字对应 Preview 论文，不包含后续模型版本的变化。[模型配置 §4.2.1](https://arxiv.org/html/2606.19348v1#S4.SS2.SSS1)、[预训练 §1](https://arxiv.org/html/2606.19348v1#S1)

## CSA、HCA 与缓存容量

CSA 先以 4 倍压缩生成 KV，再由 Lightning Indexer 选择 Top-k 压缩条目。压缩使用重叠窗口，每项聚合最多 8 个原始位置；Pro 的 1024 项对应约 8K 个位置的聚合覆盖量，不代表保留 8192 个相互独立、逐 Token 可精确检索的 KV。HCA 以 128 倍压缩生成 KV，在全部已压缩条目上执行密集注意力；每层另有 128 Token 局部窗口，保留近邻细节。[CSA §2.3.1](https://arxiv.org/html/2606.19348v1#S2.SS3.SSS1)、[HCA §2.3.2](https://arxiv.org/html/2606.19348v1#S2.SS3.SSS2)

生产低精度方案将主 KV 的 RoPE 维度存为 BF16，其余维度存为 FP8；CSA 索引器的 QK 路径采用 FP4。论文图 1 的百万上下文比较给出 Pro 推理计算量约为 V3.2 的 27%、缓存约为 10%；Flash 分别约为 10% 和 7%。推理计算量以等效 FP8 FLOPs 计，图 1 的缓存缩小倍数分别为 9.5 和 13.7；这些是估算值，不是整套服务的实测显存。[效率讨论 §2.3.4](https://arxiv.org/html/2606.19348v1#S2.SS3.SSS4)、[FP4 量化训练 §5.2.1](https://arxiv.org/html/2606.19348v1#S5.SS2.SSS1)、[图 1](https://arxiv.org/html/2606.19348v1#S0.F1)

### 同口径复算

固定官方配置给出：Pro 的 61 个骨干层含 30 层 CSA、31 层 HCA；配置末尾另一个压缩比为 0 的条目属于 MTP，不计入骨干层。V3.2 为 61 层，MLA 潜变量维度 512、RoPE 维度 64、索引器维度 128。[Pro 配置](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/resolve/5607980f3a4b8ea0371b9f11e1848ac41f14979e/inference/config.json)、[Pro 模型实现](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/resolve/5607980f3a4b8ea0371b9f11e1848ac41f14979e/inference/model.py)、[V3.2 配置](https://huggingface.co/deepseek-ai/DeepSeek-V3.2/resolve/a7e62ac04ecb2c0a54d736dc46601c5606cf10a6/config.json)

按论文及固定官方源码列出的低精度格式，复算骨干网络缓存有效载荷；数值与图1一致，参考模拟缓冲区和完整服务显存另计：

- Pro 主 KV 每条为 $(512-64)\times1+64\times2=576$ 字节；CSA 索引 KV 每条为 $128\times0.5=64$ 字节。
- V3.2主KV的512维latent采用FP8，64维RoPE采用BF16，主KV为 $512\times1+64\times2=640$ 字节；128维FP8索引再加128字节，合计768字节。固定源码第566–573行分别处理latent与RoPE，并明确实际部署的latent使用FP8、参考缓冲区模拟量化后转回BF16；第453–454行索引缓存另存FP32缩放因子。本表剔除这些缩放元数据。（[V3.2固定缓存实现](https://huggingface.co/deepseek-ai/DeepSeek-V3.2/resolve/a7e62ac04ecb2c0a54d736dc46601c5606cf10a6/inference/model.py)）
- 长度为 $L$ 时，$C_{\mathrm{Pro}}=30\lfloor L/4\rfloor(576+64)+31\lfloor L/128\rfloor576$，$C_{\mathrm{V3.2}}=61L(640+128)$。

| 上下文长度 | Pro 有效载荷 | V3.2 有效载荷 | Pro / V3.2 |
| --- | --- | --- | --- |
| 1,000,000 Token | 4.939 GB，等于 4.600 GiB | 46.848 GB | 10.544% |
| 1,048,576 Token，即图 1 的 1024K | 5.179 GB，等于 4.824 GiB | 49.124 GB | 10.544% |

这里 GB 为 $10^9$ 字节，GiB 为 $2^{30}$ 字节。后一行缩小约 9.484 倍，与图 1 的 9.5 倍标注一致；摘要的 10% 是取整表达。复算不计量化缩放因子、滑动窗口、未完成压缩的状态、分页填充、MTP 和服务运行开销，不能直接作为 GPU 显存预算。**4.6 GiB 的分母在这组条件下是约 46.848 GB，而非 15 GB**。

官方简化推理代码的 `Attention.kv_cache` 和 `Indexer.kv_cache` 使用 BF16；代码注释明确说明 FP8 模拟及参考缓存格式。因此，直接按该实现分配的缓冲区计算，会得到不同于生产低精度格式的容量。在 1,000,000 Token 时，仅 Pro 骨干主 KV 与索引 KV 的 BF16 有效载荷约为 9.848 GB。[参考推理实现](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/resolve/5607980f3a4b8ea0371b9f11e1848ac41f14979e/inference/model.py)

缓存管理还包含滑动窗口、未达到压缩长度的尾部状态，以及 CSA/HCA 的分页布局；论文将固定状态缓存与可增长的压缩 KV 缓存分开管理。[缓存结构 §3.5.1](https://arxiv.org/html/2606.19348v1#S3.SS5.SSS1)

## mHC 与优化器

mHC 将残差流扩展为四个通道，对残差混合矩阵施加双随机约束，并用 20 次 Sinkhorn-Knopp 迭代近似归一化。理想双随机矩阵的谱范数不超过 1，且矩阵乘积仍为双随机矩阵，约束的是残差混合路径。有限次数迭代与有限精度是工程近似；这一性质不等于整个包含非线性子层的模型处处非膨胀，也不保证训练没有异常。[mHC §2.2](https://arxiv.org/html/2606.19348v1#S2.SS2)、[模型设置 §4.2.1](https://arxiv.org/html/2606.19348v1#S4.SS2.SSS1)

Muon 用于大部分模块；词嵌入、预测头、RMSNorm 权重以及 mHC 的静态偏置和门控因子使用 AdamW。Hybrid Newton-Schulz 共进行 10 次迭代：前 8 次使用 $(3.4445,-4.7750,2.0315)$，后 2 次使用 $(2,-1.5,0.5)$，提高近似正交化精度。分布式实现采用 BF16 随机舍入传输 MoE 梯度，并在本地用 FP32 累加，兼顾带宽与数值精度。[优化器 §2.4](https://arxiv.org/html/2606.19348v1#S2.SS4)、[分布式实现 §3.4.1](https://arxiv.org/html/2606.19348v1#S3.SS4.SSS1)

## 训练稳定性与 OPD

Anticipatory Routing 将特征计算与路由索引的参数时间解耦：当前步骤使用当前参数计算特征，路由索引使用历史参数预先计算并缓存。系统仅在检测到 Loss Spike 时回滚并临时启用该机制，稳定后恢复常规训练；启用期间额外耗时约 20%，论文报告全程摊销开销可忽略。SwiGLU 截断在 Pro 和 Flash 的预训练全程使用：线性分支限定于 $[-10,10]$，门控分支上界为 10。训练效果是作者报告的经验观察，不能据此推导必然消除所有训练失稳。[训练稳定性 §4.2.3](https://arxiv.org/html/2606.19348v1#S4.SS2.SSS3)

后训练先分别训练领域专家，再将能力蒸馏到统一学生模型，以 OPD 替代最终混合 RL 阶段。学生生成自己的轨迹，各领域教师在这些轨迹上提供分布；损失采用学生对教师的反向 KL，并在完整词表上计算 Logit 蒸馏，减少采样近似的梯度方差。领域专家自身仍通过微调与 RL 训练，不能把“替代混合 RL”理解为完全取消 RL。工程系统支持十余个教师模型的调度。[后训练 §5.1](https://arxiv.org/html/2606.19348v1#S5.SS1)、[OPD §5.1.2](https://arxiv.org/html/2606.19348v1#S5.SS1.SSS2)、[教师调度 §5.2.2](https://arxiv.org/html/2606.19348v1#S5.SS2.SSS2)

## 基准成绩与条件

| 项目 | 论文结果 | 解释边界 |
| --- | --- | --- |
| Codeforces | Pro-Max 3206；GPT-5.4 xHigh 3168；Gemini-3.1-Pro High 3052 | 作者的内部评测：14 场 2025 年 5—11 月 Div.1、114 题；每题生成 32 个解答，再抽取 10 个组成提交序列，按比赛排名估算 Rating，非单次提交成绩 |
| Putnam-2025 | Proof-perfect 120/120 | 混合形式化与非形式化推理、形式化验证及大规模计算扩展；非单次回答或固定小预算成绩 |
| MRCR 1M | Pro-Max 83.5；Opus-4.6 Max 92.9；Gemini-3.1-Pro High 76.3 | 指标为 MMR；128K 以上出现性能下降，不等于所有长上下文任务都同幅度下降 |
| Apex / Apex Shortlist | Pro-Max 38.3 / 90.2 | 两个不同测试集的 Pass@1，不能混为同一成绩 |

以上数字分别见 [§5.3 及表 6、7](https://arxiv.org/html/2606.19348v1#S5.SS3)、[Putnam 图 8](https://arxiv.org/html/2606.19348v1#S5.F8)、[MRCR 图 9](https://arxiv.org/html/2606.19348v1#S5.F9)。评测结果反映整模型在规定框架和预算下的表现，单项成绩不能单独证明 CSA、mHC 或 Muon 的因果贡献。

## 工程基础设施

- **DeepGEMM 与 MegaMoE**：融合 MoE 的计算、通信和访存，推进批次不变性与确定性计算；[论文 §3.1](https://arxiv.org/html/2606.19348v1#S3.SS1)、[官方 MegaMoE 合入记录](https://github.com/deepseek-ai/DeepGEMM/pull/304)。
- **TileLang**：用于融合注意力、索引器及完整词表 OPD 等算子，以数值语义和编译优化兼顾开发效率与性能；[论文 §3.2](https://arxiv.org/html/2606.19348v1#S3.SS2)。
- **3FS 与 DSec**：3FS 支持分布式数据及镜像读取，DSec 提供 Agent 后训练与评测沙箱；[官方 3FS](https://github.com/deepseek-ai/3FS)、[沙箱 §5.2.5](https://arxiv.org/html/2606.19348v1#S5.SS2.SSS5)。
- **训练与推理框架**：在流水线、重计算、上下文并行及缓存管理中支持压缩注意力与 mHC；[论文 §3.4](https://arxiv.org/html/2606.19348v1#S3.SS4)、[§3.5](https://arxiv.org/html/2606.19348v1#S3.SS5)。

## 来源时间与引用范围

Bilibili 官方视频接口记录 `BV153oRBXEsG` 的作者为唐国梁Tommy、时长 1029 秒、`pubdate=1777126260`，对应上海时间 **2026-04-25 22:11:00**。arXiv 记录技术报告 v1 于 **2026-04-26 14:49:33 UTC** 提交，对应上海时间 **2026-04-26 22:49:33**。视频发布时间比 arXiv 提交早 24 小时 38 分 33 秒；这两个元数据不支持“视频直接依据后来提交的 arXiv v1”这一来源关系。模型技术结论由论文独立支持，视频作为讲解来源引用。[视频官方接口](https://api.bilibili.com/x/web-interface/view?bvid=BV153oRBXEsG)、[arXiv 提交历史](https://arxiv.org/abs/2606.19348v1)
