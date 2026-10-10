---
title: 模型推理优化：Continuous Batching 与 Ragged Batching
source: https://www.bilibili.com/video/BV1Nqp46GETj
author: 张司机在路上
published: 2026-10-09
ingested: 2026-10-09
updated: 2026-10-09
tags:
  - AI
  - Continuous Batching
  - Ragged Batching
  - KV Cache
  - Attention
  - 推理优化
  - 模型工程
  - 资料摘要
---

# 模型推理优化：Continuous Batching 与 Ragged Batching

原始资料：[[raw/sources/模型工程/推理与服务/Continuous Batching如何实现高并发推理]]

## 核心结论

一次只送一个请求时，GPU 的大部分算力会空转。Batching 把多个输入放进同一次前向，以提高吞吐。朴素批处理按最长请求补齐，并且要等整批都输出 EOS 才释放；先结束的位置空耗算力，新请求只能等待。Continuous Batching 在每一轮前向结束时摘掉已结束请求、补入等待请求，因此同一轮可以同时包含 Prefill 和 Decode。

不等长输入若升维后补齐，会把短的 Decode 请求填充到最长提示词；资料认为这种填充不可取，改为首尾相接。LayerNorm、QKV 线性映射和 FFN 按 Token 独立，不需要知道请求边界。Attention 必须用 Ragged Batching 把点积限制在同一请求内。（[[raw/sources/模型工程/推理与服务/Continuous Batching如何实现高并发推理#不等长输入不要先补齐|两种拼法]]）

## 教学示例中的形状

作者的例子里，D 的 Prefill 输入是 $[4, d]$，B 和 C 的 Decode 输入各是 $[1, d]$。补齐方案得到 $[3, 4, d]$，拉平方案得到 $[6, d]$。KV Cache 中，D 是 0 个旧 Token 加 4 个新 Token，B 是 4 加 1，C 是 2 加 1。Ragged Attention Kernel 分别计算 $Q(4)\times K(4)$、$Q(1)\times K(5)$ 和 $Q(1)\times K(3)$，再拼回 $[6, d]$。

`cu_seqlens = [0, 4, 5, 6]` 对应 Token 数 $[4, 1, 1]$，用来从合并矩阵切出每个请求。KV 按物理页分块读入，块内用在线 Softmax 累加。这些数字都属于该教学示例，不是某个框架版本的固定配置。（[[raw/sources/模型工程/推理与服务/Continuous Batching如何实现高并发推理#用边界表把合并矩阵切回每个请求|边界与分块]]）

## 与已有机制的边界

本资料点名 vLLM、SGLang 实现了连续批处理，但没有展开二者的调度器源码，也不能把 [[wiki/sources/模型推理优化：SGLang RadixAttention|RadixAttention]] 的前缀树等同于按轮换人。分块读取沿用物理页，分页布局本身见 [[wiki/sources/模型推理优化：PagedAttention 分页、前缀共享与驱逐|PagedAttention]]。在线 Softmax 只被当作已介绍过的累加方法，算法步骤仍以 [[wiki/sources/模型推理优化：FlashAttention 算子融合、在线 Softmax 与 Tiling|FlashAttention]] 为准。

分数图说“最后一行与 Value 加权得到输出 Token 向量”，随后的 Kernel 又把 D、B、C 的 Query 行拼成六行输出。两处都保留为视频自己的讲法，不把其中一处改写成另一处。

## 适用边界

- 吞吐提升是本资料对 batching 目的的说明，没有给出加速比，也不表示单个请求的等待一定变短。
- $[4, d]$、$4\times4$ 下三角、$1\times5$ 和 `cu_seqlens` 都是作者示例。图中 FFN 激活函数标为 ReLU，旁白只说“激活函数”，不能据此认为所有模型的 FFN 都使用 ReLU。
- 伪代码展示的是讲解用的 `ragged_batching` 流程，不是 vLLM 或 SGLang 某一版本的完整内核。

## 核验与整理记录

- 官方标题为《Continuous Batching如何实现高并发推理》，作者“张司机在路上”，发布于 2026-10-09，单 P，官方时长 603 秒。播放地址时长 602.766 秒。官方合集为“Codex工作原理”，当前列在正片最后一条；视频没有宣布期数，因此按无编号技术专题归档，不命名为第几期。
- 官方播放器接口返回字幕轨与章节均为空。核验级别为原始音轨：602.767 秒音轨的辅助转写覆盖到 602.76 秒，主题顺序与本地字幕一致。辅助转写只用于发现缺口和同音字，不作为新事实来源。
- 本地字幕 272 条，范围 00:00:00,040–00:10:02,740。结尾比音轨短 0.027 秒，没有越过视频结尾。字幕相邻间隔都不超过 0.8 秒，辅助转写也没有超过 1.2 秒的空档。未发现重复旁白，没有删除任何时间段。
- 画面字幕和示意图确认 Tensor Core、vLLM、SGLang、Prefill、Decode、EOS、Naive Batching、Continuous Batching、Ragged Batching、KV Cache、`cu_seqlens = [0, 4, 5, 6]`、LayerNorm、QKV Linear、Attn Out Linear、ReLU 和形状 $[4, d]$、$[3, 4, d]$、$[6, d]$。据此校正字幕里的同音误写。开头画面出现一篇题为 Continuous batching 的 Hugging Face 文章，旁白没有引用其作者或论据，正文不引入该页内容。
- 去掉口播自指、看图提示和互动语，保留早高峰类比、A/B/C/D 替换顺序、两种拼法、Token 独立计算与 Attention 例外、分数图的最后一行讲法，以及 Kernel 的六行输出。没有把教学形状写成框架的通用性能结论。

## 关联

- [[wiki/sources/模型推理优化：PagedAttention 分页、前缀共享与驱逐]]
- [[wiki/sources/模型推理优化：FlashAttention 算子融合、在线 Softmax 与 Tiling]]
- [[wiki/sources/模型推理优化：SGLang RadixAttention]]
- [[wiki/sources/模型推理优化：KV Cache 与 Prompt Cache 的复用层级]]
- [[wiki/sources/模型架构：KV Cache 显存公式与 MHA、MQA、GQA]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
