---
title: 模型推理优化：FlashAttention 算子融合、在线 Softmax 与 Tiling
source: https://www.bilibili.com/video/BV1TU8x6hEVb
author: 张司机在路上
published: 2026-08-23
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - FlashAttention
  - Attention
  - GPU
  - HBM
  - SRAM
  - Online Softmax
  - Tiling
  - 推理优化
  - 模型工程
  - 资料摘要
---

# 模型推理优化：FlashAttention 算子融合、在线 Softmax 与 Tiling

原始资料：[[raw/sources/模型工程/推理与服务/FlashAttention为什么又快又节省GPU显存？|FlashAttention为什么又快又节省GPU显存？]]

## 核心结论

FlashAttention 仍计算标准 Attention 的全部 $n\times n$ 分数，主要收益来自减少 HBM I/O 和避免在 HBM 中物化完整分数与概率矩阵。Kernel Fusion 把点积、Softmax 和乘 $V$ 串成一个流程，Online Softmax 使最大值、分母与输出可以稳定地边扫描边更新，Tiling 再把单行递推扩展为多行分块并行。

## 存储约束

资料以 A100 为例：片上 SRAM 约 20 MB、带宽约 19 TB/s，HBM 为 40 GB、带宽约 1.5 TB/s。$Q$、$K$、$V$ 和 $O$ 是 $n\times d$，中间分数与概率矩阵是 $n\times n$；整块矩阵无法进入 SRAM，但单个 Token 或小块向量可以。标准实现的主要等待因此来自各阶段之间反复写入和读取 HBM。

## 在线递推

截至第 $i$ 项的最大值和 Softmax 分母为：

$$
m_i=\max(m_{i-1},x_i)
$$

$$
d_i'=d_{i-1}'e^{m_{i-1}-m_i}+e^{x_i-m_i}
$$

归一化输出同时递推：

$$
o_i'=o_{i-1}'\frac{d_{i-1}'e^{m_{i-1}-m_i}}{d_i'}+\frac{e^{x_i-m_i}}{d_i'}v_i
$$

最大值变化时，修正因子把已经累计的分母与输出换到新基准，无需重新读取此前所有元素。分块实现把 $m$、$d$ 和 $o$ 从标量或向量扩展为多行状态，每个分数 Tile 在 SRAM 中消费后立即丢弃。

## 证据边界

- 资料引用的 GPT-2 Benchmark 为 Hugging Face 10 天、NVIDIA Megatron-LM 5 天、FlashAttention 2.5 天，并称模型质量相同；这些数字只对应论文中的特定训练配置。
- 资料称“显存读写”从 35 GB 降到 4.4 GB，但没有定义该指标是峰值占用、累计 I/O 还是其他口径，不能自行改写。
- A100 的 SRAM、HBM 与 CPU DRAM 数字是教学示例，不代表其他 GPU 或系统。
- 本资料讲解单行递推与前向 Tiling 原理，没有展开反向传播、因果 Mask、Dropout、不同数据类型或具体内核版本。

## 关联

- Attention 与 QKV：[[wiki/sources/模型架构：多头注意力与 QKV]]
- KV Cache 显存：[[wiki/sources/模型架构：KV Cache 显存公式与 MHA、MQA、GQA]]
- KV 物理分页：[[wiki/sources/模型推理优化：PagedAttention 分页、前缀共享与驱逐]]
- GPU 带宽与计算：[[wiki/sources/AI 计算硬件：内存带宽、互联与软件生态]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
