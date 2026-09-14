---
title: FlashAttention为什么又快又节省GPU显存？
source: https://www.bilibili.com/video/BV1TU8x6hEVb
author: 张司机在路上
created: 2026-08-23
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
---

# FlashAttention为什么又快又节省GPU显存？

FlashAttention 计算的是标准 Attention，并没有省略 $n\times n$ 个注意力分数中的任何一项。它的主要优化对象不是乘法数量，而是数据在 GPU 的 HBM 与片上 SRAM 之间的搬运：通过算子融合、Online Softmax 和 Tiling，让中间结果尽量在 SRAM 中计算和消费，不再把完整的注意力分数矩阵写入 HBM。

## 论文基准给出的结果

资料引用 FlashAttention 论文的一组 Benchmark：同样训练 GPT-2，Hugging Face 实现需要 10 天，NVIDIA Megatron-LM 需要 5 天，FlashAttention 需要 2.5 天，所得模型质量相同。资料还称“显存读写”从 35 GB 降至 4.4 GB，约缩小 8 倍。

“35 GB 到 4.4 GB”在资料中被称为显存读写，但视频没有说明它表示峰值占用、累计 I/O 量还是其他显存指标，因此本文保留来源名称，不改写为更具体的指标。资料发布时还把 FlashAttention 类算法称为现代大模型技术栈中的默认选择，并列举 PyTorch 2、Hugging Face 和 vLLM；这是一项发布时概括，不代表这些项目的所有版本与全部 Attention 路径都固定使用同一实现。

## GPU 的存储层级

资料用 A100 展示三层存储的容量与带宽差异：

- 每个 SM 都有片上 SRAM；整卡合计约 20 MB，带宽约 19 TB/s。
- HBM 容量为 40 GB，带宽约 1.5 TB/s。
- CPU 一侧的 DRAM 容量可以超过 1 TB，示例带宽为 12.8 GB/s。

层级越高，速度越快，容量越小。资料把 HBM 比作长期仓库：模型权重、KV Cache 和各层激活值存放在这里；SRAM 是当前计算的临时工作区，只保留正在处理的小块数据，计算完成后即可释放。

在这套教学模型中，数据参与 SM 运算前要从 HBM 搬到片上 SRAM。提高速度的关键因此是让数据进入 SRAM 后完成尽可能多的计算，减少中途写回和再次读取 HBM。上述容量与带宽只对应资料展示的 A100 示例，不是所有 GPU 的固定参数。

## Attention 的中间矩阵为什么放不进 SRAM

假设上下文包含 $n$ 个 Token，每个 Head 的向量维度为 $d$，则 $Q$、$K$、$V$ 均为 $n\times d$ 矩阵：

$$
\operatorname{Attention}(Q,K,V)=\operatorname{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

资料把 $n$ 描述为可增长到十万甚至百万量级的上下文长度，把 $d$ 描述为常见取值 32、64 或 128 的 Head Dimension。$Q$、$K$、$V$ 与输出 $O$ 的形状为 $n\times d$，分数矩阵与 Softmax 概率矩阵则为 $n\times n$。这些整块矩阵无法装进示例 SRAM；单个 Token 对应的 $q$、$k$、$v$、$o$ 向量只有若干 KB，可以完整放入。

这一区分是后续推导的起点：矩阵放不下，单行或小块可以放下，因此算法需要逐行或分块处理，而不是在 HBM 中物化全部中间矩阵。

## 标准 Attention 的三个步骤

为简化表达，先把缩放因子暂时省略。Attention 可以拆成三步：

$$
S=QK^T
$$

第一个 Query 向量 $q_1$ 分别与 $k_1$ 到 $k_n$ 点积，得到第一行 $n$ 个未归一化分数。

$$
P=\operatorname{softmax}(S)
$$

Softmax 对每一行独立归一化，使该行 $p_1$ 到 $p_n$ 的和为 1。

$$
O=PV
$$

第一行输出 $o_1$ 是所有 Value 向量按注意力权重加权后的和：

$$
o_1=\sum_{j=1}^{n}p_{1j}v_j
$$

资料将标准实现概括为三个独立阶段：每一步从 HBM 读取输入，计算后把中间结果写回 HBM，下一步再读出。大量时间因此消耗在中间矩阵的写入与读取，而不只是算术运算本身。

## 算子融合先消除中间写回

Kernel Fusion 的直观目标，是把点积、Softmax 和乘 $V$ 串成一个流程。读取一个 $q$ 后，循环读取各组 $k_i$ 与 $v_i$，计算：

$$
x_i=q^Tk_i\cdot\frac{1}{\sqrt{d_k}}
$$

如果暂不考虑数值稳定性，可以分别累加 Softmax 分子对输出的贡献和分母：

$$
\tilde{o}=\sum_i e^{x_i}v_i,\qquad d=\sum_i e^{x_i}
$$

最后返回：

$$
o=\frac{\tilde{o}}{d}
$$

Softmax 的整行分母相同，因此可以在循环结束后统一相除。$x_i$、$d$ 和当前输出向量的大小都不随上下文长度 $n$ 增长，可以留在 SRAM；$K$ 和 $V$ 只需从 HBM 顺序读取一次，最后再把输出写回。

这段融合仍不能直接使用，因为 $x_i$ 来自两个向量的点积，数值范围不受控，$e^{x_i}$ 可能超过浮点数范围。

## Safe Softmax 为什么仍需多次扫描

Safe Softmax 先取整行最大值 $m=\max_i x_i$，再计算：

$$
\operatorname{softmax}(x_i)=\frac{e^{x_i-m}}{\sum_j e^{x_j-m}}
$$

分子和分母同时乘以 $e^{-m}$，结果不变。由于 $x_i-m\leq0$，指数值落在 $(0,1]$，从而避免正向溢出。

朴素实现需要扫描三遍：第一遍寻找整行最大值，第二遍计算指数并累加分母，第三遍除以分母得到概率。第二遍必须等第一遍完成，因为每一项都依赖全局最大值 $m_n$。这会破坏原本希望一次扫描完成融合的目标。

## Online Softmax 同时更新最大值与分母

Online Softmax 把全局最大值换成截至当前位置的最大值：

$$
m_i=\max(m_{i-1},x_i)
$$

同时定义前 $i$ 项在当前基准 $m_i$ 下的分母：

$$
d_i'=\sum_{j\leq i}e^{x_j-m_i}
$$

当最大值从 $m_{i-1}$ 更新为 $m_i$ 时，旧分母中的所有项都要更换指数基准。它们可以整体乘以同一个修正因子，无需重新扫描：

$$
d_i'=d_{i-1}'e^{m_{i-1}-m_i}+e^{x_i-m_i}
$$

因此，最大值与分母可以在第一遍同时算出，Safe Softmax 的三遍扫描变成两遍。不过，此时输出仍需第二遍遍历 $K$ 和 $V$。

## 输出向量也可以在线更新

要把两遍继续压成一遍，可以让当前输出同样只依赖已经扫描的前 $i$ 项：

$$
o_i'=\sum_{j\leq i}\frac{e^{x_j-m_i}}{d_i'}v_j
$$

当新的 $x_i$ 到达时，旧输出先按新的最大值与分母重新缩放，再加入当前 Value 的贡献：

$$
o_i'=o_{i-1}'\frac{d_{i-1}'e^{m_{i-1}-m_i}}{d_i'}+\frac{e^{x_i-m_i}}{d_i'}v_i
$$

实现中先保存旧分母 $d_{old}$，更新 $m$ 与 $d$，再构造：

$$
\operatorname{rescale}=\frac{d_{old}e^{m_{old}-m}}{d}
$$

并更新：

$$
o\leftarrow o\cdot\operatorname{rescale}+\frac{e^{x-m}}{d}v
$$

扫描结束时，$o$ 已经是最终输出。$x$、$m$、$d$ 与 $o$ 全程留在 SRAM，不需要把分数、概率或其他中间结果写回 HBM。这里的递推关系使点积、稳定 Softmax 与 Value 加权真正融合为一次扫描。

## Tiling 让 SRAM 与 Tensor Core 得到利用

逐个处理 $q$、$k$、$v$ 便于推导，却无法充分利用 SRAM 容量和 Tensor Core。实际 FlashAttention 会把 $Q$、$K$、$V$ 切成 Tile：外层循环一次把一批 Query 搬入 SRAM，内层循环按块扫描 $K$ 和 $V$。

标量形式的最大值 $m$ 与分母 $d$ 随之变成按多行维护的向量，单个输出向量 $o$ 也变成一批输出组成的矩阵。每次算出一个小块分数后，就在 SRAM 中立即更新这一批行的 $m$、$d$ 和 $O$，使用完成后丢弃小块分数；所有 $K$、$V$ Tile 扫描结束后，才把最终输出 Tile 写回 HBM。

Tiling 没有改变前述递推原理，只是把单行版本扩展为多行并行版本，使有限 SRAM 和 Tensor Core 的计算能力得到更充分利用。

## 三项技术解决的是数据搬运

FlashAttention 在资料中被归纳为三项关键技术：

1. **Kernel Fusion**：把计算分数、Softmax 和乘 $V$ 从三个 HBM 往返合并为一个流程，中间结果不写回 HBM。
2. **Online Softmax**：用最大值、分母和输出的修正因子实现边扫描边更新，是稳定融合能够成立的前提。
3. **Tiling**：从一次搬运一个向量扩展为一次搬运一个矩阵小块，利用 SRAM 容量与 Tensor Core 并行算力。

算法仍然计算完整的 $n\times n$ 注意力分数，没有降低标准 Attention 的分数数量。节省的是 HBM 与 SRAM 之间的额外传输，以及完整中间矩阵的 HBM 存储。现代 GPU 上的优化很多时候并非少做乘法，而是减少等待访存。
