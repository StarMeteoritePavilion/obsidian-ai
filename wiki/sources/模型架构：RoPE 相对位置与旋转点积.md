---
title: 模型架构：RoPE 相对位置与旋转点积
source: https://www.bilibili.com/video/BV1kqYe6DEvF
author: 张司机在路上
published: 2026-09-13
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Transformer
  - Attention
  - 位置编码
  - RoPE
  - 模型架构
  - 模型原理
  - 资料摘要
---

# 模型架构：RoPE 相对位置与旋转点积

原始资料：[[raw/sources/模型原理/模型架构/RoPE 旋转位置编码比正弦位置编码好在哪里|RoPE 旋转位置编码比正弦位置编码好在哪里]]

## 核心结论

正弦位置编码与 RoPE 都利用正弦、余弦建立相对位置结构，但进入 Attention 的方式不同。正弦编码在投影前与 Embedding 相加，点积展开后产生语义—位置交叉项，投影矩阵也会作用于位置编码；RoPE 在投影后旋转 Query 和 Key，使两个绝对旋转合并为只依赖 $n-m$ 的相对旋转。

$$
(q'_m)^Tk'_n=(W_Qx_m)^TRoPE(n-m)(W_Kx_n)
$$

因此，本资料所说的“更好”指向 Attention Score 的代数形式：最终结果直接依赖 Query、Key 和相对位置，而不分别依赖两个绝对位置。

## 两条推导链

正弦位置编码自身满足：

$$
PE_i(pos+k)=R_i(k)PE_i(pos)
$$

$$
PE_i(m)^TPE_i(n)=\cos\bigl(\omega_i(m-n)\bigr)
$$

但在 $q_m=W_Q(x_m+PE(m))$、$k_n=W_K(x_n+PE(n))$ 的形式下，$q_m^Tk_n$ 会展开为四项，其中两项混合单个绝对位置，最后一项还夹有 $W_Q^TW_K$。

RoPE 则使用 $q'_m=RoPE(m)(W_Qx_m)$ 与 $k'_n=RoPE(n)(W_Kx_n)$。由于 $RoPE(m)^T=RoPE(-m)$、$RoPE(-m)RoPE(n)=RoPE(n-m)$，点积只留下相对位置。

## 证据边界

- 视频的结论来自上述公式推导，没有提供准确率、长文本外推或运行效率对照，不能据此认定 RoPE 在所有模型和任务上都具有统一幅度的性能优势。
- DeepSeek V3、GLM-4.5 和 Qwen3 是资料发布时列举的使用实例，不代表它们的具体 RoPE 配置完全相同。
- 本资料讨论位置怎样进入 Query、Key 点积，不讨论 Value、Attention Mask、缩放、Softmax 或多头间差异，不能把它当成完整 Attention 实现说明。

## 关联

- [[wiki/sources/模型架构：正弦位置编码与注意力的顺序缺口]]
- [[wiki/sources/模型架构：多头注意力与 QKV]]
- [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- [[wiki/sources/模型架构：KV Cache 显存公式与 MHA、MQA、GQA]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
