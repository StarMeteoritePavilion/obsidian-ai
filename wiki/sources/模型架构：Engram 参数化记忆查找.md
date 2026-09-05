---
title: 模型架构：Engram 参数化记忆查找
source: https://www.bilibili.com/video/BV1hLwMzwEVx
author: 唐国梁Tommy
published: 2026-03-15
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型架构
  - DeepSeek
  - Engram
  - MoE
  - 资料摘要
---

# 模型架构：Engram 参数化记忆查找

原始资料：[[raw/sources/模型原理/模型架构/DeepSeek Engram：用"查字典"打败Transformer，推理能力反而涨了5分？|DeepSeek Engram：用“查字典”打败 Transformer]]

## 核心结论

Engram 在 Transformer 中增加参数化 N-gram 查找通道，把局部模式召回与 MoE 的动态计算分开。词表投影统一大小写和前导空格等表面形式，资料称词表规模因此压缩约 23%；多尺度 N-gram 与多头哈希降低单次碰撞影响，上下文门控过滤多义和碰撞噪声，约 16K 参数的深度可分离卷积连接相邻查找结果，再通过残差注入主干。资料转述的消融实验显示，移除词表压缩或上下文门控都会使性能明显下降。Engram 随模型端到端训练，不是外部非参数化 RAG。

这项工作增加了第二种稀疏轴：MoE 按 Token 选择少量计算专家，Engram 按输入确定的地址选择少量记忆表项。前者主要控制活跃计算，后者把一部分参数容量放到可预取的查找结构中。

## 系统与模型协同

查表地址只依赖输入 Token，CPU 可以在 GPU 计算浅层时预取后续需要的表项。资料所述实验把 100B 参数的 Engram 表放在 CPU 内存，使用 H800 推理 8B 模型，吞吐从每秒 6,315 Token 降至 6,140 Token，下降 2.8%。这一结果依赖对应硬件、模型和流水线，不能作为其他部署的固定成本。

Engram 的层位置也同时受传输与建模约束：太浅时缺少掩盖 PCIe 传输的计算时间，太深时局部特征注入价值下降。资料给出的选择是在 Layer 2 之后。

## 实验结果与解释边界

MoE-27B 与 Engram-27B 都保持 26.7B 总参数和 3.8B 激活参数；Engram 版本把路由专家从 72 个减至 55 个，把 5.7B 参数转给查表记忆。资料转述的增量为 MMLU +3.4、CMMLU +4.0、BBH +5.0、ARC-Challenge +3.7、HumanEval +3.0 个百分点；多针大海捞针从 84.2 提高到 97.0，变量追踪从 77.0 提高到 89.0。

LogitLens 与 CKA 分析显示，Engram 模型的中间层 Loss 更早下降，第 5 层表示与基线第 12 层高度相似。资料据此解释为浅层更早完成局部模式处理，后续层释放给复杂推理；这属于机制解释，不表示能够直接读出每层的固定功能。

Engram-40B 把记忆表扩展到 18.5B 参数，总参数为 39.5B，激活参数仍为 3.8B。固定预算实验的较优区域约为 75%～80% 稀疏参数给 MoE、20%～25% 给 Engram。作者进一步提出“20% 记忆、80% 动态计算”的解释，但不同领域是否采用同一比例仍待验证。

视频还提出直接改写 Embedding 表项以实现低成本 Model Editing，以及把查表机制扩展到多模态向量 Patch。两项均为后续设想，不是本资料已经验证的能力。

## 关联

- MoE 的稀疏计算：[[wiki/sources/模型架构：MoE 稀疏专家路由]]
- Token、Attention 与模型内部表示：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
- 外部非参数化检索：[[wiki/sources/上下文工程：第四期 RAG 检索增强生成]]
- HBM、DDR5、PCIe 与参数搬运：[[wiki/sources/AI 计算硬件：内存带宽、互联与软件生态]]
