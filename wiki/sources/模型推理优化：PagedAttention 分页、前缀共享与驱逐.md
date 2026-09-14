---
title: 模型推理优化：PagedAttention 分页、前缀共享与驱逐
source: https://www.bilibili.com/video/BV1go836fECf
author: 张司机在路上
published: 2026-08-18
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - PagedAttention
  - vLLM
  - KV Cache
  - Prompt Caching
  - 推理优化
  - 模型工程
  - 资料摘要
---

# 模型推理优化：PagedAttention 分页、前缀共享与驱逐

原始资料：[[raw/sources/模型工程/推理与服务/PagedAttention帮vLLM把分页机制搬进大模型|PagedAttention帮vLLM把分页机制搬进大模型]]

## 核心结论

PagedAttention 将 KV Cache 切成固定大小的物理 Block，再用每个请求的 Block Table 把连续逻辑块映射到分散的 GPU 块。它不再为每个请求预留最大连续显存，也不要求请求使用的物理块相邻，因而缩小内部碎片和外部碎片。

相同完整前缀通过链式 Block Hash 命中同一批物理块。命中后无需复制 KV，只需让两个 Block Table 指向同一 `KVCacheBlock` 并增加 `ref_cnt`；未命中的块才分配新物理空间并执行 Prefill。

## 核心数据结构

- `free_block_queue` 在 CPU 内存中组织空闲块元数据，每个节点是 `KVCacheBlock`。
- `KVCacheBlock` 保存 `block_id`、`ref_cnt`、`_block_hash`、`prev_block` 和 `next_block`。
- `BlockHashToBlockMap` 是全局前缀哈希到物理块的映射，供所有请求查询。
- `req_to_blocks` 为每个请求保存按逻辑顺序排列的物理块列表。
- GPU 中每层单个物理块的 Tensor 形状为 $[2,\text{block\_size},\text{num\_kv\_heads},\text{head\_dim}]$。

Block Hash 链接前一块哈希和当前块 Token，因此命中表示从开头到当前块的 Token 序列一致。资料指出只有填满的 Block 才计算哈希，避免为仍可被 Decode 填充的末块提前生成稳定键。

## 回收不等于作废

当 `ref_cnt` 降为 0 时，物理块返回 `free_block_queue`，但其 KV 内容与 Block Hash 映射仍保留。后续请求在该块被重新分配前仍可命中它。只有物理块将被新内容覆盖时，旧哈希才会 Evict；资料中的驱逐不是定时清理或请求结束即过期。

`ref_cnt` 负责共享安全，Block Hash 负责按完整前缀找块，Block Table 负责恢复每个请求的逻辑顺序，空闲队列则负责物理块的再分配。四者共同支撑不复制的前缀 KV 共享。

## 适用边界

- 资料讲解的是 vLLM 的 PagedAttention 数据结构，不是 OpenAI 服务端的可见实现。
- 视频用 Codex `cached_tokens` 按 512 递增的抓包现象引出固定分块，但没有证明 OpenAI 的物理 Block 大小为 512 Token，也没有证明 Codex 后端直接使用 vLLM。
- 资料所称 vLLM 通常使用 16 Token 的 Block 是来源口径，没有绑定 vLLM 版本或配置，不应视为所有环境的固定值。

## 关联

- [[wiki/sources/模型推理优化：FlashAttention 算子融合、在线 Softmax 与 Tiling]]
- [[wiki/sources/模型推理优化：Codex 自动前缀缓存]]
- [[wiki/sources/模型架构：KV Cache 显存公式与 MHA、MQA、GQA]]
- [[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
