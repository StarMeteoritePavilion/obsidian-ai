---
title: Continuous Batching如何实现高并发推理
source: https://www.bilibili.com/video/BV1Nqp46GETj
author: 张司机在路上
created: 2026-10-09
tags:
  - AI
  - 推理优化
  - 模型工程
---

# Continuous Batching如何实现高并发推理

大模型推理时，如果一次只把一个请求送进 GPU，大部分算力会空转。这像早高峰的道路：每辆车明明至少能拉四个人，实际上只坐了司机，一整条路的运力大部分被浪费。GPU 上发生的也是这件事，它是推理引擎中 batching 的出发点。批处理把多个输入同时送进模型一起计算，让 Tensor Core 的计算资源被充分利用，从而使吞吐量大幅提升。vLLM、SGLang 这些推理框架在此之上进一步实现了 Continuous Batching，也就是连续批处理。

## 一次回答的两个阶段

生成一次回答分为两个阶段。Prefill（预填充）先把整段提示词在一次前向中读入，为接下来的回答做准备。Decode（解码）在准备好之后开始输出：每次前向只能生成一个 Token，再把这个 Token 作为新输入，添加到已有提示词中，计算下一个 Token。这种生成方式叫做 autoregressive，即自回归。为了便于理解，这里描述的过程只考虑一个请求。

## 朴素批处理要等整批结束

同时收到请求 A、B、C 时，最简单的做法是按最长的那条请求，把三者填充成同样长度的张量，再送进 Transformer。每一轮前向同时输出三个请求的下一个 Token。

某个请求已经输出 EOS、序列结束后，整个批处理仍不能结束。三个请求都要继续计算，直到所有请求都输出结束，整个 batch 才算处理完。朴素批处理（Naive Batching）因此既同时开始，也同时结束，都得等最长的那条请求推理完毕，才能够整批释放。

如果有一个请求的输出特别长，先结束的位置只能空耗算力，新请求也只能干等，GPU 资源被白白浪费。

## 连续批处理按轮换入请求

Continuous Batching 仍用同一个例子。推理时发现请求 A 已经输出 EOS，就不再让 A 留在 batch 中，而是让新来的请求直接替换进来。例如请求 D 是用户输入的四个 Token 的提示词，下一轮送入 Transformer 的就是四个 D Token、一个 B Token 和一个 C Token。

下一步若请求 C 也生成完毕，下一个等待的请求会替换原来 C 的位置。请求调度的时机从整批全部结束，变成每一轮前向计算结束：结束一个，就立刻补上一个正在等待的请求。GPU 不再空转，新请求也不用等整批跑完。

同一轮 batch 里因此会同时出现正在 Prefill 的请求和正在 Decode 的请求。

## 不等长输入不要先补齐

假设 embedding 的维数是 $d$。请求 D 正在做 Prefill，输入是 $[4, d]$；请求 B 和请求 C 都在做 Decode，输入都是 $[1, d]$。

第一种合并方法是升维，把每个输入从二维矩阵变成三维。例如 $[4, d]$ 变成 $[1, 4, d]$。三个矩阵长度不同，只能把两个 $[1, d]$ 填充成 $[4, d]$，合并后的输入是 $[3, 4, d]$。如果 batch 里进来一个提示词很长的请求，其他 Decode 请求都要填充到同样长度才能处理，这是巨大的计算浪费，所以填充的方法不可取。

第二种方法不升维，直接把三个矩阵首尾相连、拉平，变成 $[6, d]$。这样没有任何填充数据，但后续计算会带来新问题。

## 除了 Attention，其余计算按 Token 独立

Decoder 一层的结构是：输入先经过 LayerNorm 做归一化，进入多头注意力，再与输入做一次残差相加；然后再次经过 LayerNorm，进入 FFN 前馈网络，最后又是一次残差相加，得到这一层的输出。图中的多头注意力标为 Masked Multi-Head Attention。

多头注意力把输入通过线性映射投影成 Query、Key、Value 三个矩阵，然后计算注意力分数，再对分数做一次线性变换。图中这一路径是 QKV Linear、Attention 和 Attn Out Linear。FFN 比较简单：做一次线性变换，先升维，经过一个激活函数，再线性变换降维回去。图中的激活函数是 ReLU。

LayerNorm、QKV 线性映射和整个 FFN，也就是除 Attention 以外的这些计算，都是 Token 与 Token 之间独立的。每个 Token 各算各的：每一行向量变换成结果中的另一行，与上下其他 Token 的向量没有关系。这个性质适合拼接起来的输入，因为这些计算完全不关心每个 Token 属于哪个请求，每个 Token 都做同样的变换。

Attention 完全不同。它的本质是每个 Token 的 Query 向量要和别的 Key 向量互相做点积。拼接后的输入如果直接做 Attention，请求 D 里的 Token 与前面的 Token 做点积没有问题，与后面的 Token 做点积则不行，因为那些 Token 根本不属于同一个请求。从 D 的 Token 去访问 B 和 C 的 Token 是不对的。

因此，除 Attention 以外，这些计算对于合并输入都是无感的；只有 Attention 不能直接套用，必须分清合并输入里哪些 Token 属于哪些请求。

## Ragged Batching 只在本请求内做点积

让 Attention 处理合并请求的方法叫做 Ragged Batching，也就是不规则批处理。图中的注意力分数里，Query 就是这一轮输入做一次映射得到的结果。之前的输入是六行，Query 矩阵也是六行。与它们做点积的 Key、Value 则不一样。

请求 D 在做 Prefill，它的 Key 和 Value 也由输入做线性变换得到，所以是四个 Token。两两做点积后，注意力分数是一个 $4\times 4$ 的下三角矩阵。请求 B 在做 Decode，输入的 Query 要和它之前出现过的所有 Token 做点积；那些 Token 缓存在 KV Cache 里，取出来以后再加上当前输入的一个 Token，Key、Value 一共包含五个 Token。请求 B 的注意力分数是一个 $1\times 5$ 的矩阵。请求 C 也是 Decode，对应一个 Query 和三个 Key。

对所有请求计算完分数以后，按该图的讲法，把分数的最后一行和 Value 矩阵加权求和，就得到最终输出的 Token 向量。两个 Token 不属于同一个请求时，注意力分数就是 0。图上大部分分数都是 0，实际上并不会真正计算它们，画出来只是为了看清哪些值才需要计算。

在 Ragged Batching 里，注意力矩阵的行数和列数变得不一样。Query 是本轮新 Token 拼起来的，Key 是 KV Cache 里的历史 Token 再加上新 Token，所以 Query 和 Key 的长度根本不必相同。

## 用边界表把合并矩阵切回每个请求

算子的数据流从合并输入 $[6, d]$ 进入，先经过 QKV Linear，得到 Query、Key、Value。Key 和 Value 送进 KV Cache，按不同请求分别合并：请求 D 的缓存是空的，写入 0 个旧 Token 加 4 个新 Token；请求 B 把 1 个新 Token 合并进已经有的 4 个 Token；请求 C 是 2 个旧 Token 加 1 个新 Token。KV Cache 写好以后，把各请求的 Key 和 Value 读出来，与 Query 一起送进 Ragged Attention Kernel，按请求分开并行计算。D 的分数是四个 Query 和四个 Key 做点积，B 是一个 Query 对五个 Token 做点积，C 是一个 Query 和三个 Key 做点积。得到的输出再拼回六行，也就是 $[6, d]$，然后经过 Attn Out Linear。

右侧伪代码里有一张边界表 `cu_seqlens`，记录每个请求在合并输入中从第几行开始、到第几行结束。三个请求的 Token 数分别是 4、1、1，对应的 `cu_seqlens` 是 $[0, 4, 5, 6]$。第 0 个请求从第 0 行到第 $4-1$ 行。有了这张表，就可以在合并的大矩阵里访问每一个请求的 Token。

伪代码先一次性把输入做投影，得到 Q、K、V。然后对 batch 里的所有请求，用对应新增的 K 和 V 更新该请求的 KV Cache。做完以后开始计算注意力：根据 `cu_seqlens` 把这个请求自己的那几行 Query 切出来，再去读它自己的那一块 KV Cache。缓存按物理页分页存放，所以一块一块读进来。每达到一块，就让这几行 Query 和这一块 Key 做一次点积，得到注意力分数，再用这个分数加权这一块 Value，把结果累加进一个状态里面。这里用的是在线 Softmax；此前的 FlashAttention 讲解已经完整介绍过它的代码。

所有请求算完，输出矩阵又是完整的六行，最后再过一遍输出投影，就得到 Attention 这一层的计算结果。
