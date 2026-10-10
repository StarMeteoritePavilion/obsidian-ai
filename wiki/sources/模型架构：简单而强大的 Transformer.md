---
title: 模型架构：简单而强大的 Transformer
source: https://www.bilibili.com/video/BV1NCgVzoEG9?p=6
author: 飞天闪客
published: 2025-07-21
ingested: 2026-10-10
updated: 2026-10-10
tags:
  - AI
  - Transformer
  - 注意力
  - 多头注意力
  - 模型原理
  - 资料摘要
---

# 模型架构：简单而强大的 Transformer

原始资料：[[简单而强大的 Transformer]]

## 核心结论

这一集用翻译说明 Transformer。全连接网络没有词与词的上下文，长度还得一一对应。RNN 能保留顺序，但串行，而且长句会丢失长期依赖。新结构先给每个词加位置编码，再用可训练的 $W_Q$、$W_K$、$W_V$ 得到 $Q$、$K$、$V$。一个词的 $Q$ 与各个 $K$ 做点积，当作权重去加对应的 $V$。这就是注意力。只算一组相似度不够灵活，例子改成两个头，各组分别计算再拼接，称为多头注意力。

前面的讲解省掉了缩放、掩码和 Softmax。结构图再把它们补上，并在拼接后加一次输出线性变换。残差和归一化被说成缓解梯度消失、稳定分布，没有展开。位置编码怎么计算，也明确留到以后。

编码器处理原文，解码器生成译文。解码器先用掩码挡住还没生成的词，再拿自己的 $Q$ 去看编码器的 $K$、$V$。线性层投到词表，Softmax 给出下一个词的分布，取最大的一项。GPT 被说成只保留解码器：不翻译，只根据前面的词猜下一个词。

## 画面里的数字和式子

词向量维度在这个例子里假设为 6。句子是 “I love you baby”，译文是「我爱你宝贝」。两个头的示意给每个词画出两套 $q$、$k$、$v$。

注意力和多头的式子是

$$
\mathrm{Attention}(Q,K,V)=\mathrm{softmax}\left(\frac{QK^{T}}{\sqrt{d_k}}\right)V
$$

$$
\mathrm{MultiHead}(Q,K,V)=\mathrm{Concat}(\mathrm{head}_1,\ldots,\mathrm{head}_h)W^{O}
$$

$\mathrm{head}_i=\mathrm{Attention}(QW_i^{Q},KW_i^{K},VW_i^{V})$。结构图把全连接块标成 Feed Forward，重复次数写成 $N\times$，没有给出 $N$。一次教学分布是「一」0、「的」0、「爱」0.7、「喜」0.2。这些都是讲解用的数，不是一次真实模型输出。

## 与其他资料的边界

[[wiki/sources/模型架构：从词嵌入到 RNN|第 5 P]] 只把 Transformer 点名为摆脱按时间步传递的方案，没有写 $Q$、$K$、$V$。本集才展开。它没有重讲 RNN 的 $W_{xh}$、$W_{hh}$、$W_{hy}$。

[[wiki/sources/模型架构：多头注意力与 QKV|多头注意力与 QKV]] 和 [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支|编码器与解码器]] 来自另一组资料。本集的两个头、维度 6 和 0.7／0.2 不能并进那些资料的头数或采样设置。本集取概率最大的一项，没有讲 Temperature 或 Top-K。图上的 $\sqrt{d_k}$ 没有配数值例子。

[[wiki/sources/模型架构：正弦位置编码与注意力的顺序缺口|正弦位置编码]] 才计算位置编码。本集只说先加上，算法留着。残差为什么能缓解梯度消失，也没有推导。

GPT 在这里只指解码器一半，不是 [[wiki/sources/模型架构：Linear、Activation 与 MLP|GPT-2 XL]] 的 1600 维，也不是另一系列里 768、3072、768 的配置。

## 核验与整理记录

- 官方视频标题为《【闪客】一小时从函数到 Transformer》，作者“飞天闪客”，发布于 2025-07-21。全片 8 P，总时长 4157 秒。字幕对应第 6 P「06 简单而强大的 Transformer」。播放器确认 `page_no` 为 6。该 P 整数时长 607 秒，播放信息 606.310 秒，音轨 606.311 秒。官方字幕轨和章节均为空。第 1–5 P 已另文收录，没有期数冲突。
- 核验级别为原始音轨。用户字幕 252 条，范围为 00:00:00,040–00:10:05,440，SHA-256 为 `165d2b2e5a219eb7f3d703bed507cb6f313d59471fba89b722a0aa67595dfcbf`。辅助转写覆盖到片尾，末句与字幕一致；字幕结束后约 0.9 秒没有新旁白。文本去重后没有重复旁白，没有删除任何时间段。
- 00:05:25.780–00:05:28.280 的间隔里，音轨补上用户字幕没写的“上面的讲解过程”。两侧说的是先前省略了缩放、掩码和 Softmax。另外两处 1.3–1.9 秒的间隔是鼓掌之后和“总的来说”之前，没有新句子。
- 完成帧确认点积、两个头的 $QKV$、Scaled Dot-Product Attention、Mask、Softmax、$\sqrt{d_k}$、Concat 与 $W^{O}$、Feed Forward、$N\times$，以及概率表「爱」0.7、「喜」0.2。字幕“那GPT的底层其实就是transform的一半”把用户字幕里的 GBT 定为 GPT。用户字幕里的“点击”“磁向量”“QQV”“此表”“I字”分别改为点积、词向量、$QKV$、词表和「爱」。
- 论文页出现在结尾，旁白没有读出层数或 $d_{\mathrm{model}}$，正文不写入这些数。位置编码算法和残差推导都保持未展开。弹幕征集和加油口播没有写入正文。

## 关联

- [[wiki/sources/模型架构：从词嵌入到 RNN]]：上一 P 只点名 Transformer。
- [[wiki/sources/模型架构：多头注意力与 QKV]]
- [[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- [[wiki/sources/模型架构：正弦位置编码与注意力的顺序缺口]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
