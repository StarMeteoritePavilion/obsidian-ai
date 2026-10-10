---
title: 模型推理优化：SGLang RadixAttention
source: https://www.bilibili.com/video/BV1e9h969EJk
author: 张司机在路上
published: 2026-09-26
ingested: 2026-09-26
updated: 2026-10-10
tags:
  - AI
  - SGLang
  - RadixAttention
  - Radix Tree
  - KV Cache
  - Prompt Caching
  - 推理优化
  - 模型工程
  - 资料摘要
---

# 模型推理优化：SGLang RadixAttention

原始资料：[[raw/sources/模型工程/推理与服务/SGLang的核心注意力机制 RadixAttention]]

## 核心结论

SGLang 使用 Radix Tree 组织不同请求的公共前缀。树节点保存连续 Token 段及其 KV Cache 物理槽位下标；命中公共前缀后，系统跳过该部分 Prefill，只计算未命中的后缀。节点分割只修改下标，不移动 GPU 中的 KV 数据。

作者所述布局为每层预分配 K、V 缓存；`free_pages` 另存空闲槽位下标，本身不是第三类 KV 数据。显存不足时，示例把叶子按 `last_access_time` 放入最小堆，优先删除最久未访问的叶子并释放其槽位。（[[raw/sources/模型工程/推理与服务/SGLang的核心注意力机制 RadixAttention#物理槽位与空闲池|缓存与索引的区别]]）

## 案例中的关键因果

四次请求依次使用“为什么我的代码会报错”“为什么显存老是不够用”“为什么我的代码这么慢”“为什么我的代码不能编译”。第一次占用槽位 1–5；第二次拆出“为什么”，复用 1 并新增 6–9；第三次再拆出“我的代码”，复用 1–3 并新增 10–11；第四次无需分割已有边，但需要淘汰旧叶子以获得槽位。

作者在时刻 1 将两个叶子的访问时间都设为 1；时刻 2 分割后，“会报错”和“这么慢”两叶子的访问时间为 2，“显存老是不够用”仍为 1。因此第四次请求先删除后者，归还 6–9，再使用 6、7。若省略分割后的时间变化，就无法完整解释淘汰顺序。（[[raw/sources/模型工程/推理与服务/SGLang的核心注意力机制 RadixAttention#四个时刻中的匹配、分割与淘汰|完整四时刻示例]]）

## 与 PagedAttention 的关系

两篇资料介绍的前缀复用都减少重复 Prefill，但前缀组织方式不同：[[wiki/sources/模型推理优化：PagedAttention 分页、前缀共享与驱逐|PagedAttention 资料]]介绍固定 Block 的链式哈希查表；本资料介绍沿 Radix Tree 的 Token 段寻找最长公共前缀。作者称两种方法复杂度相同，但没有给出复杂度推导或同条件性能测量，不据此判断两者端到端速度相同。

## 适用边界

- 本资料只说明视频展示的 RadixAttention 数据结构与示例，不扩展为所有 SGLang 版本的完整实现说明。
- Token 切分、槽位编号和访问时间是作者的教学示例，不能直接外推为任意模型的分词结果或所有版本的实现行为；本资料也未展开并发请求使用中的节点保护条件。

## 关联

- [[wiki/sources/模型推理优化：PagedAttention 分页、前缀共享与驱逐]]
- [[wiki/sources/模型推理优化：Continuous Batching 与 Ragged Batching]]：SGLang 在该资料中被列为连续批处理的实现者之一；前缀树复用与按轮换入请求不是同一机制。
- [[wiki/sources/模型推理优化：KV Cache 与 Prompt Cache 的复用层级]]
- [[wiki/sources/模型架构：KV Cache 显存公式与 MHA、MQA、GQA]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
