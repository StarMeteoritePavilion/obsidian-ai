---
title: 模型推理：从 Token、Latent 到多模态交错思维
created: 2026-09-03
updated: 2026-09-04
tags:
  - AI
  - 模型原理
  - 模型推理
  - 综合
---

# 模型推理：从 Token、Latent 到多模态交错思维

模型推理不能只用“生成下一个 Token”或“在隐藏空间思考”概括。现有资料呈现出三种相互衔接、但不能混为一谈的表示层：离散 Token 是输入输出接口，Latent State 承担模型内部连续计算，ThinkMorph 的 Interleaved CoT 则把部分中间处理显式展开为交替的文本片段和图像片段。表示机制之外还存在独立的测试时计算、执行与服务层：DRAG 和 IterDRAG 分配检索文档、演示与迭代步骤，DSpark 调整草稿怎样生成、验证多少位置以及算力怎样随负载分配，计算硬件资料区分参数搬运、并行计算与卡间通信，API 成本资料则把 Prefill、Decode、KV Cache、Prompt Caching 和 Batch 映射为计费与架构选择。（[[wiki/sources/大语言模型：Token 与两类 Embedding|Token 与两类 Embedding]]、[[wiki/sources/模型原理：Token Space 与 Latent Space|Token Space 与 Latent Space]]、[[wiki/sources/多模态推理：ThinkMorph 交错思维链|ThinkMorph]]、[[wiki/sources/上下文工程：DRAG 与 IterDRAG 推理扩展|DRAG 与 IterDRAG]]、[[wiki/sources/模型推理优化：DSpark 投机解码|DSpark]]、[[wiki/sources/AI 计算硬件：内存带宽、互联与软件生态|AI 计算硬件]]、[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制|Token 成本]]）

## 三层表示的职责

| 表示层 | 主要职责 | 可见性 | 主要限制 |
| --- | --- | --- | --- |
| Token Space | 文本输入、输出、训练标签、工具调用和审计 | 人可读 | 离散化边界影响模型收到的信息结构 |
| Latent Space | 语义表示、上下文关联和内部连续计算 | 通常不可直接读 | 难解释、难审计，解码工具也可能失真 |
| 多模态交错思维链 | 文本规划与视觉操作交替推进 | 文本和生成图像可检查 | 图像 Token 成本高，并非每个任务都需要 |

三层不是互斥架构。文本和图像输入仍会被编码为离散或连续表示，模型内部仍需 Latent 计算；ThinkMorph 的不同之处，是把部分推理过程外化为文本思考、图像操作和后续文本验证，而不是把全部中间过程隐藏在 Latent State 中。

## Token ID、Token Embedding 与 RAG Embedding

Tokenizer 把文字切分并映射为固定词表中的 Token ID；Token Embedding 再把离散编号映射为连续向量，并随大语言模型共同训练。前者是训练期间固定的预处理，后者是模型参数。One-Hot 乘以线性映射与直接查 Embedding 表在数学关系上相通，但工程实现不必显式构造完整 One-Hot 向量。（[[wiki/sources/大语言模型：Token 与两类 Embedding|Token 与两类 Embedding]]）

RAG Embedding 面向整段文本，训练目标是让相关文本靠近、无关文本远离，通常还需要 Pooling 汇总多个位置。它与大语言模型内部表示都属于连续向量，却不能因此直接等同：Token Embedding 服务于模型输入，下一个 Token 预测塑造生成模型；RAG Embedding 服务于语义检索，对比学习塑造文本距离。任意 LLM Hidden State 也不能未经单独训练和处理就视为可用的检索向量。（[[wiki/sources/大语言模型：Token 与两类 Embedding|Token 与两类 Embedding]]、[[wiki/sources/上下文工程：RAG 个人知识库基础架构|RAG 个人知识库]]）

## Transformer 的三种基础路线

原始 Transformer 用 Encoder—Decoder 结构完成序列到序列转换：Encoder 处理完整输入并形成内部表示，Decoder 结合该表示和已经生成的目标 Token 继续自回归输出。GPT 路线保留适合下一个 Token 预测的 Decoder 主体，形成 Decoder-only 架构；BERT 路线使用 Encoder-only 架构，通过恢复 `[MASK]` 等目标学习双向文本表示。（[[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支|Transformer 架构导论]]）

三种路线共享 Transformer 的分层参数计算，却不能只用“理解”与“生成”两个拟人化标签区分。实际差异还包括注意力可见范围、是否接收独立 Encoder 输出、训练目标和解码方式。资料中的“含义矩阵”适合作为内部表示的教学类比，不代表模型形成了可直接读取、与人类概念一一对应的语义表。

## Linear、Activation 与 MLP 提供基础变换

Linear 通过 $y=Wx+b$ 把一个向量映射为另一个向量，Weight 与 Bias 由训练数据确定。多个 Linear 直接复合仍然是线性函数；ReLU、Sigmoid、tanh 和 GELU 等 Activation 在层间引入非线性，使 FFN／MLP 能够拟合更复杂的关系。Transformer 的 Feed Forward 模块建立在这类结构上。（[[wiki/sources/模型架构：Linear、Activation 与 MLP|Linear、Activation 与 MLP]]）

参数并不是从目标公式中直接读取。线性回归教学示例先用平方误差衡量预测与训练目标的差距，再根据损失对 $w$、$b$ 的梯度确定更新方向，由 Learning Rate 控制步长；多个样本的平方误差取平均后形成 MSE，Batch Size 决定一轮使用的样本数。该示例说明参数怎样更新，不表示所有模型都使用同一损失函数、批量或学习率。（[[wiki/sources/模型训练：梯度下降与均方误差|梯度下降与均方误差]]）

PyTorch 手写数字训练实例把抽象更新过程映射为工程链路：Dataset 在返回单个样本时执行 `transform`，DataLoader 组成批次，模型产生 Logits，CrossEntropyLoss 计算分类损失，`backward()` 求梯度，Optimizer 清空梯度并更新参数，`state_dict()` 则负责保存和恢复权重。示例使用训练集样本展示预测，只能证明对应样本的结果；模型泛化仍需测试集验证。（[[wiki/sources/模型训练：PyTorch 手写数字识别实战|PyTorch 训练实战]]）

GPT-2 XL 的输出 Linear 将 1600 维内部表示映射为 50257 个 Token 的匹配分数。视频演示的三层 MLP 则采用 `1→128→256→1`，包含 33537 个可训练参数，以 2000 条数据拟合非线性曲线。这些实例说明同一基础模块可以承担不同映射职责，不表示模型参数能够逐项翻译为人类概念。

## 注意力怎样形成上下文表示

Token 进入 Transformer 后先映射为 Embedding，再分别投影为 Query、Key 和 Value。$QK^T$ 计算当前位置与其他位置的匹配分数，按 $\sqrt{d_k}$ 缩放并经过 Softmax 后形成注意力权重，最后对 Value 加权求和。自回归模型还用因果 Mask 把未来位置的权重压到 0，保证当前位置只读取已经出现的 Token。（[[wiki/sources/模型架构：多头注意力与 QKV|多头注意力]]）

多头结构让多组独立投影并行学习不同关系，再拼接各头结果。资料用“语法表”“需求表”和“内容矩阵”解释 Key、Query 与 Value，但明确这些只是教学类比；真实隐向量维度与 Attention Head 通常不能直接命名为可读概念。这条链路解释 Latent State 怎样吸收上下文，不意味着人类能够逐维读出模型内部含义。

## Attention Residuals 沿深度选择表示

标准残差连接让原始输入与各模块输出沿深度连续相加，为训练信号提供直通路径；固定单位权重也会使深层隐藏表示的数值持续累积，并稀释单层贡献。Attention Residuals（AttnRes）把 Attention 的动态聚合从 Token 维度移到层深度：当前层使用可训练 pseudo-query 生成 Softmax 权重，再选择性汇总此前表示。（[[wiki/sources/模型架构：Attention Residuals 层间选择性聚合|Attention Residuals]]）

Full AttnRes 访问所有此前层输出，提供细粒度选择，但需要保存和读取完整层历史；Block AttnRes 在块内保留标准残差，只在块间执行注意力聚合，以较粗粒度降低开销。它与 Token 间 Attention 共享“根据相关性加权”的思想，但处理对象分别是序列位置与网络深度，不能把两者视为同一个注意力轴。

## MoE 把容量与活跃计算分开

Attention 形成上下文表示后，FFN 负责继续变换每个位置。Dense 模型让所有 FFN 参数处理每个 Token；MoE 把大型 FFN 拆为多个专家，由 Router 为当前 Token 选择少量路由专家，再与共享专家的结果加权组合。总参数因而表示模型容纳的专家容量，激活参数则更接近单次 Token 实际使用的计算规模。（[[wiki/sources/模型架构：MoE 稀疏专家路由|MoE 稀疏专家路由]]）

DeepSeekMoE 对比中，145B MoE 有 144.6B 总参数、22.2B 激活参数，每 4K Token FLOPs 为 585.6T；67B Dense 的总参数和激活参数均为 67.4B，对应 2057.5T FLOPs。资料据此概括 MoE 的计算优势，但该表没有直接测量端到端延迟或 API Token 价格，不能用 FLOPs 代替这两项指标。

## 交错推理之前的多模态架构

ViT 提供了从像素到视觉特征的基础入口。资料中的 $224\times224$ 彩色图片先被切成 196 个 $16\times16$ 图块，每块展开为 768 个 RGB 数字；局部模型提取图块特征后，Transformer 编码器通过无因果限制的注意力关联全图，输出融合上下文的图块表示。这里的“狗眼睛”“狗尾巴”等名称只是对数字特征的教学类比，不能视为模型内部存在可直接读取的自然语言标签。（[[wiki/sources/多模态模型：ViT 图像分块与编码|ViT 图像分块与编码]]）

DLSS／FSR 资料展示了视觉模型的另一种输入组织方式：系统不只处理一张图片，还利用亚像素抖动得到的历史帧，并以 Motion Vector 对齐移动物体，再加入 Z-buffer、曝光值和 Reactive Mask 等渲染侧信号。单帧视觉编码回答“当前图像包含什么”，时序重建则回答“怎样用多帧与几何信息恢复当前高分辨率画面”；二者都使用数字视觉表示，但任务、输入和输出不同。资料称 DLSS 4 在 2025 年由 CNN 切换为 Vision Transformer，这不等于所有 ViT 都用于游戏超分辨率。（[[wiki/sources/图像重建：DLSS 与 FSR 的时序超分辨率|DLSS 与 FSR]]）

多模态推理首先受输入表示约束。典型系统由视觉编码器、模态接口和预训练语言模型组成；MLP Projection、Q-Former 与 Cross-Attention 分别以直接投影、固定 Query 压缩和按需跨模态注意力连接视觉与语言。资料对超过 120 个模型的汇总中，输入分辨率从 224 提高到 336 的画面结果为提升 11.5%，接口从 MLP 换为 Q-Former 为提升 1.2%。这组特定比较说明视觉细节是否进入模型可能比接口形式更先构成瓶颈，但不能外推为所有任务的架构排名。（[[wiki/sources/多模态模型：架构、数据、推理与检索|多模态技术地图]]）

MCoT 的能力还取决于训练路线和数据质量。资料将其分为 Prompt 提示、SFT 长链训练和 RL 三阶段；最后一阶段只有在结果可以可靠验证时才容易形成有效奖励。数学、科学和代码可借助答案或执行结果验证，情感、创意和社会常识则缺少稳定的单一评分标准。ThinkMorph 展示的是统一模型怎样显式交替生成文本和图像，二者共同说明“推理表示”与“训练反馈”是两项相互作用但不可混同的设计选择。（[[wiki/sources/多模态模型：架构、数据、推理与检索|多模态技术地图]]、[[wiki/sources/多模态推理：ThinkMorph 交错思维链|ThinkMorph]]）

《Thinking with Visual Primitives》进一步指出，视觉信息已经进入模型也不等于推理能够稳定引用它。自然语言中的“左边那个”或“他旁边的”在复杂场景中可能发生指代漂移；点和边界框可以作为中间推理变量，把语言概念绑定到可重复引用的图像坐标。它解决的是 Reference Gap，而非单纯增加感知分辨率。（[[wiki/sources/多模态推理：视觉原语与 Reference Gap|视觉原语专题]]）

视觉原语要成为可靠推理变量，还需要数据、训练与验证共同约束。报告把 97,984 个原始数据源经过语义和视觉几何过滤缩减为 31,701 个，再采样、去重形成超过 4,000 万个样本；框与点分别训练专家后，通过 Unified RFT 和 OPD 合并。格式、质量和任务准确性三层奖励进一步检查坐标语法、原语—答案一致性、迷宫合法探索与双向路径匹配。表示形式因此只提供“可以怎样思考”的接口，训练信号才决定模型是否会稳定使用该接口。（[[wiki/sources/多模态推理：视觉原语的数据、训练与奖励|视觉原语训练专题]]）

## 推理表示应跟随任务需要

纯文本 CoT 适合抽象规划、逻辑计算和可审计表达，但难以直接验证局部视觉细节。Latent Reasoning 可以减少必须写成自然语言的中间步骤，却把解释和追责压力转移到 Decoder、Probe、SAE 或因果干预工具。Interleaved CoT 在文本与视觉空间同时搜索，适合需要裁剪、放大、定位或视觉重构的任务，但生成图像的成本明显更高。

因此，表示方式应由信息需求决定：语言和已有视觉编码足以解决问题时，纯文本路径更短；必须产生新的视觉证据时，加入图像操作；需要压缩或并行保留多个内部方向时，才考虑更多 Latent 计算。ThinkMorph 的自主模式切换与 Token-Latent Hybrid 的设想都指向同一原则：保留可读接口，把额外计算放在确实能增加信息的表示空间中。（[[wiki/sources/模型原理：Token Space 与 Latent Space|Token 与 Latent]]、[[wiki/sources/多模态推理：ThinkMorph 交错思维链|ThinkMorph]]）

## 可见思维链与内部计算不是同一对象

可见 CoT 是模型在 Token Space 中生成的文本，内部计算则发生在不可直接读取的 Latent State 中。前者可以帮助人检查步骤，却不能自动成为后者的忠实记录。1776 年案例中，模型正确叙述闰年规则后给出相反结论；DataAlchemy 的任务泛化实验还出现了错误推理过程与正确答案并存的情况，后者可由两种变换在实验设置中的可交换性解释。（[[wiki/sources/大语言模型：思维链的模式匹配与泛化边界|思维链泛化边界]]）

DataAlchemy 进一步从任务、长度和格式三个维度观察到：测试分布偏离训练分布时，可见推理链会变得脆弱。该结果支持思维链受到训练数据分布约束的解释，但不能据此断言全部大模型内部都不存在抽象推理。评估时应同时检查最终答案、可见步骤、二者的一致性以及任务所处的分布范围。

## 评估不能只看最终准确率

不同表示方式消耗的资源不同。比较文本 CoT、Latent Reasoning 和 Interleaved CoT 时，至少需要同时记录可见 Token、图像 Token、隐藏步骤、FLOPs、实际延迟、准确率和审计能力。Best-of-N 的收益还必须结合采样数和选择器成本；少生成文本不代表总计算更少，多生成视觉步骤也不必然带来更高准确率。

现有资料的实验数字分别来自不同模型、任务和论文设置，不能横向拼成统一排名。ThinkMorph 模式切换数据还存在讲解文字与论文图注的基准归属冲突，说明评估结论必须保留原始表格、评判器和适用条件，而不能只摘取提升比例。（[[wiki/sources/模型原理：Token Space 与 Latent Space|Token 与 Latent]]、[[wiki/sources/多模态推理：ThinkMorph 交错思维链|ThinkMorph]]）

视觉原语实验也说明最终答案准确率不足以评价中间推理。DS_Maze_Navigation 和 DS_Path_Tracing 分别比所列 GPT-5.4 结果高 16.3 与 10.2 个百分点，但本文模型在 CountQA、CV-Bench 和 OmniSpatial 上仍略低于 Gemini 3 Flash；而且比较统一使用低推理预算。点和框的收益集中在需要精确引用与连续轨迹的任务，不构成模型整体能力排名。报告也没有提供标准消融表，无法把收益分别归因于视觉原语、框点分训、OPD 或 CSA。（[[wiki/sources/多模态推理：视觉原语与 Reference Gap|视觉原语上集]]、[[wiki/sources/多模态推理：视觉原语的数据、训练与奖励|视觉原语下集]]）

## 表示机制、RAG 推理扩展与执行优化是三条轴

Token、Latent State 和 Interleaved CoT 回答的是中间信息以什么形式存在、哪些步骤对人可见；DRAG 与 IterDRAG 回答的是在有效上下文预算内，怎样把测试时计算分配给外部文档、示例和迭代检索；DSpark 回答的是自回归输出已经确定以后，怎样减少生成这些 Token 的等待时间和无效计算。投机解码仍以 Token 为输入输出接口，目标模型内部仍执行 Latent 计算，但草稿模型先预测一段 Token，目标模型再并行验证，从而在保持目标输出分布的条件下提高生成速度。（[[wiki/sources/上下文工程：DRAG 与 IterDRAG 推理扩展|DRAG 与 IterDRAG]]、[[wiki/sources/模型推理优化：DSpark 投机解码|DSpark]]）

三条轴不能用同一组指标替代。表示路线需要比较准确率、可见 Token、隐藏步骤、图像 Token 和可审计性；RAG 推理扩展还要记录检索文档数、示例数、迭代次数、有效上下文长度以及 EM、F1、准确率；执行优化则要记录接受长度、有效吞吐、每秒轮数、端到端延迟和并发负载。增加 RAG 的测试时计算可能提高答案质量，却不等于提高服务吞吐；增加单轮验证长度也不必然降低延迟。（[[wiki/sources/上下文工程：DRAG 与 IterDRAG 推理扩展|DRAG 与 IterDRAG]]、[[wiki/sources/模型推理优化：DSpark 投机解码|DSpark]]）

DSpark 的半自回归和置信度调度说明，模型质量与系统效率也不能分开优化。第一个草稿 Token 更依赖模型容量，后续位置更依赖连贯性；验证范围则取决于草稿通过概率和当前硬件批量。其 60%～85% 单用户生成速度提升来自 DeepSeek-V4 的特定线上条件，不构成其他模型或部署环境的通用收益保证。（[[wiki/sources/模型推理优化：DSpark 投机解码|DSpark]]）

## 从稀疏容量到长程工具调用

Kimi K2 Thinking 把模型容量、每 Token 计算量和部署精度作为三项不同变量。资料所述架构有 1.04T 总参数、32B 激活参数、384 个专家，每个 Token 使用 8 个路由专家与 1 个共享专家；MLA 压缩 Key-Value 状态，MoE 组件再通过 QAT 获得原生 Weight-only INT4。总参数增加不等于每次推理同比增加计算，量化也不等于减少逻辑步骤。（[[wiki/sources/大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练|Kimi K2 Thinking]]）

资料称原生 INT4 使生成速度提高约 2 倍，并支持 200～300 次连续工具调用。前者属于模型、硬件和推理实现共同决定的执行效率，后者属于 Agent 在长程轨迹中维持目标和上下文的行为能力。二者不能用同一指标替代：更快生成不会自动减少工具错误，长调用链也不证明端到端延迟或任务成功成本更低。

Kimi K2 Thinking 的 Test-Time Scaling 又增加了第四项变量：推理时允许模型使用多少思考时间和工具调用预算。它与 DRAG/IterDRAG 分配检索和演示预算、DSpark 优化 Token 验证执行属于不同实现，但都说明模型评估必须同时报告计算预算、工具环境和停止条件，不能只比较最终分数。

## 从推理阶段到 API 成本

模型的计算量只有映射到实际硬件数据流后，才会变成吞吐与延迟。CPU 可以执行模型所需运算，但单请求自回归生成可能先受参数读取带宽限制；HBM 提高带宽，批量请求复用参数后，瓶颈又会移向并行计算核心。模型超过单卡显存时，推理主要沿模型分片传递中间激活；大规模训练还需跨并行组同步与参数同量级的梯度，因此需要 NVLink、Infinity Fabric 等高速互联。峰值 FLOPs、显存容量、内存带宽和互联带宽回答的是不同问题，不能互相替代。（[[wiki/sources/AI 计算硬件：内存带宽、互联与软件生态|AI 计算硬件]]）

硬件执行还受到软件栈约束。PyTorch 经 cuBLAS、cuDNN 等中间层调用 CUDA 内核，长期算法适配把硬件优势放大为生态优势；ROCm 尝试兼容既有路径，TPU／OpenXLA 则另建计算、互联和软件体系。这些路线说明，推理优化不仅是模型算法问题，也受算子覆盖、框架集成、部署工具和迁移成本影响。

Prefill 与 Decode 的计算形态解释了输入和输出 Token 为什么常被区别定价。Prefill 面对完整输入，可以较为并行地建立中间状态；Decode 按自回归顺序逐 Token 生成，每一步都需要新的计算和调度。Reasoning Token、图像 Token 和音频 Token 则把用户不可见的内部生成或非文本输入继续折算为计量单位。（[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制|Token 成本专题]]）

KV Cache、Prompt Caching 和 Batch 分别作用于不同环节。KV Cache 保存当前请求已经计算的 Key 与 Value，用显存换取历史状态复用；Prompt Caching 识别跨请求重复的稳定前缀，把重复上下文变成低成本输入；Batch 允许延后请求并合并调度，用等待时间换取 GPU 利用率。三者不能互相替代，也不能只用“减少 Token 数”概括。

推理性能最终还要落到任务经济性。每百万 Token 单价忽略了重试、延迟、错误、人工稽核、运维和成功率；便宜模型若反复失败，完成任务的总成本可能更高。模型路由、缓存友好的 Prompt 结构和 Token FinOps 因此属于推理服务架构，而不只是采购或提示词技巧。

## 资料链

- [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- [[wiki/sources/大语言模型：Token 与两类 Embedding]]
- [[wiki/sources/模型原理：Token Space 与 Latent Space]]
- [[wiki/sources/大语言模型：思维链的模式匹配与泛化边界]]
- [[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- [[wiki/sources/模型训练：梯度下降与均方误差]]
- [[wiki/sources/模型训练：PyTorch 手写数字识别实战]]
- [[wiki/sources/模型架构：多头注意力与 QKV]]
- [[wiki/sources/模型架构：Attention Residuals 层间选择性聚合]]
- [[wiki/sources/模型架构：MoE 稀疏专家路由]]
- [[wiki/sources/多模态推理：ThinkMorph 交错思维链]]
- [[wiki/sources/多模态模型：ViT 图像分块与编码]]
- [[wiki/sources/图像重建：DLSS 与 FSR 的时序超分辨率]]
- [[wiki/sources/多模态模型：架构、数据、推理与检索]]
- [[wiki/sources/多模态推理：视觉原语与 Reference Gap]]
- [[wiki/sources/多模态推理：视觉原语的数据、训练与奖励]]
- [[wiki/sources/大语言模型：Kimi K2 Thinking 的 MoE 架构与 Agent 训练]]
- [[wiki/sources/上下文工程：DRAG 与 IterDRAG 推理扩展]]
- [[wiki/sources/模型推理优化：DSpark 投机解码]]
- [[wiki/sources/AI 计算硬件：内存带宽、互联与软件生态]]
- [[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
