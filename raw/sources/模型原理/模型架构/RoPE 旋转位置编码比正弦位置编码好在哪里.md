---
title: RoPE 旋转位置编码比正弦位置编码好在哪里
source: https://www.bilibili.com/video/BV1kqYe6DEvF
author: 张司机在路上
created: 2026-09-13
tags:
  - AI
  - Transformer
  - Attention
  - 位置编码
  - RoPE
  - 模型架构
  - 模型原理
---

# RoPE 旋转位置编码比正弦位置编码好在哪里

资料发布时列举的 DeepSeek V3、GLM-4.5 和 Qwen3 都使用 RoPE（Rotary Position Embedding，旋转位置编码）。RoPE 与正弦位置编码都建立在不同频率的正弦、余弦函数上，但它们把位置信息带入 Attention 的方式不同：正弦位置编码在投影前与 Embedding 相加，RoPE 则在投影后旋转 Query 和 Key。

这种差异决定了最终 Attention 点积能否直接写成 Token 语义与相对位置的函数。

## 注意力更需要相对位置

以“张三打李四谁疼”为例，“打”和“疼”的相对间隔始终为 3。即使整句话前面增加了上千个 Token，这两个 Token 的绝对位置同时后移，它们之间的语义关系与相对距离仍然没有改变。

资料据此提出理想形式：Attention Score 应由两个 Token 的语义表示及其相对距离决定，而不应因为二者在整段文本中的绝对位置同时变化而改变。

## 正弦位置编码本身具有相对位置结构

正弦位置编码把每两个维度组成一对不同频率的正弦、余弦分量。令第 $i$ 对分量的频率为：

$$
\omega_i=\frac{1}{10000^{2i/d}}
$$

一对分量可以理解为单位圆上的指针。位置 `pos` 每增加 1，指针就旋转 $\omega_i$；位置增加 $k$，则旋转 $k\omega_i$。相应的旋转矩阵为：

$$
R_i(k)=
\begin{bmatrix}
\cos(\omega_i k)&-\sin(\omega_i k)\\
\sin(\omega_i k)&\cos(\omega_i k)
\end{bmatrix}
$$

因此，只要知道当前位置的编码，就能在不知道绝对位置 `pos` 的情况下，通过 $R_i(k)$ 得到相距 $k$ 的编码：

$$
PE_i(pos+k)=R_i(k)PE_i(pos)
$$

两个位置编码分量的点积也只保留位置差。对位置 $pos_1$ 与 $pos_2$：

$$
PE_i(pos_1)^TPE_i(pos_2)=\cos\bigl(\omega_i(pos_1-pos_2)\bigr)
$$

完整位置向量的点积，就是各维度对上述余弦值的求和。可见正弦位置编码自身已经包含良好的相对位置性质；问题出现在它与 Embedding 相加并经过 Query、Key 投影之后。

## 加法怎样把绝对位置带进 Attention

设位置 $m$、$n$ 的 Token Embedding 分别为 $x_m$、$x_n$。正弦位置编码先与 Embedding 相加，再通过投影矩阵：

$$
q_m=W_Q(x_m+PE(m))
$$

$$
k_n=W_K(x_n+PE(n))
$$

令 $W=W_Q^TW_K$，将 Attention 中的点积展开：

$$
\begin{aligned}
q_m^Tk_n
={}&x_m^TWx_n
+x_m^TWPE(n)\\
&+PE(m)^TWx_n
+PE(m)^TWPE(n)
\end{aligned}
$$

第一项只包含两个 Token 的语义表示。中间两项分别混合语义与位置编码，把绝对位置 $m$、$n$ 带入最终分数。第四项看起来对应两个位置编码的点积，但中间夹着投影矩阵 $W$；只有在 $W$ 恰好是单位矩阵的特殊情况下，才能直接还原为原始位置编码只依赖相对距离的点积。

产生这些项的原因有两个：位置编码通过加法进入 Embedding，展开后必然出现交叉项；位置编码在投影之前加入，投影矩阵也会作用于它。

## RoPE 在投影之后旋转 Q 和 K

RoPE 改变两处运算：不再把位置向量与 Embedding 相加，而是使用旋转矩阵相乘；不在 Query、Key 投影之前加入位置，而是在投影完成后分别旋转 $q$ 与 $k$。

对于一个 $d$ 维向量，RoPE 将相邻两个元素分为一组，共 $d/2$ 组。第 $i$ 组在位置 `pos` 使用以下矩阵：

$$
RoPE(pos,i)=
\begin{bmatrix}
\cos(\omega_i pos)&-\sin(\omega_i pos)\\
\sin(\omega_i pos)&\cos(\omega_i pos)
\end{bmatrix}
$$

每组二维向量都按 $\omega_i pos$ 旋转。位置分别为 $m$ 与 $n$ 时，先由 Embedding 得到未经位置旋转的 Query、Key，再执行：

$$
q'_m=RoPE(m)(W_Qx_m)
$$

$$
k'_n=RoPE(n)(W_Kx_n)
$$

## 点积只留下相对位置

把旋转后的 Query、Key 代入点积：

$$
\begin{aligned}
(q'_m)^Tk'_n
={}&(W_Qx_m)^TRoPE(m)^TRoPE(n)(W_Kx_n)\\
={}&(W_Qx_m)^TRoPE(n-m)(W_Kx_n)
\end{aligned}
$$

这里使用了两个旋转矩阵的性质：

$$
RoPE(m)^T=RoPE(-m)
$$

$$
RoPE(-m)RoPE(n)=RoPE(n-m)
$$

几何上也可以这样理解：同时把两个已经旋转的向量反向旋转 $m\omega_i$，二者夹角和点积不会改变。Query 回到原方向，Key 则留下 $(n-m)\omega_i$ 的净旋转。最终点积只包含未经位置旋转的 Query、Key 与相对位置 $n-m$，不再出现分别依赖 $m$、$n$ 的交叉项。

## 两种编码的差异

正弦位置编码是向量，通过加法作用于投影前的 Embedding；RoPE 是旋转矩阵，通过乘法作用于投影后的 Query 和 Key。二者自身都具有消去共同绝对位置的数学性质，但带入真实 Attention 后结果不同。

在资料给出的推导中，正弦位置编码会因加法展开和投影矩阵产生绝对位置交叉项；RoPE 则把两个绝对旋转合并为 $RoPE(n-m)$，使 Attention 点积直接表达为 Token 语义与相对距离的函数。这是 RoPE 相对于正弦位置编码的核心结构优势。
