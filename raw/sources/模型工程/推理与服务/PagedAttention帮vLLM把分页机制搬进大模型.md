---
title: PagedAttention帮vLLM把分页机制搬进大模型
source: https://www.bilibili.com/video/BV1go836fECf
author: 张司机在路上
created: 2026-08-18
tags:
  - AI
  - PagedAttention
  - vLLM
  - KV Cache
  - Prompt Caching
  - 推理优化
  - 模型工程
---

# PagedAttention帮vLLM把分页机制搬进大模型

模型推理时，每个请求都需要在 GPU 显存中保存 KV Cache。请求会生成多长往往无法预先确定；若为每个请求预留整段最大空间，会浪费未使用的显存，若按需分配大小不一的连续空间，又会在请求结束后留下分散空洞。PagedAttention 将操作系统的分页思路应用到 KV Cache，把逻辑上连续的 Token 块映射到分散的物理显存块。

## 传统连续分配的两种碎片

内部碎片（Internal Fragmentation）来自过度预留。服务在请求刚开始时不知道最终输出长度，最直接的做法是按最大上下文长度预分配一段连续显存。实际未用满的部分依然被占用，形成块内浪费。

外部碎片（External Fragmentation）来自不同大小空间的反复分配和释放。显存中的多个空闲区域总和可能达到 1 GB，但若没有任何一段连续空间达到 1 GB，新请求仍无法放入。

PagedAttention 用等大的小块同时缩小两类问题：任何空闲物理块都可以分配给任何请求，逻辑上相邻的块不必在物理上连续；未填满的空间只会留在最后一个块内。资料称 vLLM 中一个 Block 通常包含 16 个 Token，因此按该口径，单段提示词只会在末块留下不足一个 Block 的空闲位置。

## 逻辑 Block 与物理 Block 分离

操作系统分页把程序的连续虚拟地址映射到分散的物理页。PagedAttention 采用相同的抽象：提示词按 `block_size` 切成逻辑 Block，Block Table 记录每个逻辑块落在哪个物理 KV Block。

当 `block_size = 4` 时，`ABCD` 和 `EFGH` 是两个逻辑块。它们可以分别放入物理 Block 5 和 Block 0。逻辑编号与物理编号没有数值关系，顺序由 Block Table 恢复。另一个请求若具有相同的 `ABCD` 前缀，也可以让自己的 Block Table 指向同一个物理 Block 5，无需复制 KV。

## CPU 元数据与 GPU 物理块

vLLM 启动时会把可用显存预先切成固定大小的物理 Block。空闲块的元数据在 CPU 内存中通过双向队列 `free_block_queue` 组织：分配时从队列取块，请求不再使用时将块还回队列。

队列中的每个节点是 `KVCacheBlock` 对象，包含四类信息：

- `block_id`：对应 GPU 物理块编号。
- `ref_cnt`：当前有多少个请求正在共享该块。
- `_block_hash`：该块及其完整前缀的链式哈希。
- `prev_block` 和 `next_block`：将空闲块连成双向队列的指针。

真正的 KV Tensor 位于 GPU。Transformer 的 $L$ 层 Decoder 需要独立 KV，因此同一 `block_id` 在每层都对应同一位置的切片，形成贯穿全部层的物理块。每层中单个块的 Tensor 形状为：

$$
[2,\ \text{block\_size},\ \text{num\_kv\_heads},\ \text{head\_dim}]
$$

其中 2 表示 Key 和 Value。所有块、所有层的 KV 元素总数为：

$$
L\times\text{num\_blocks}\times\text{block\_size}\times2\times\text{num\_kv\_heads}\times\text{head\_dim}
$$

这与按总 Token 数计算 KV Cache 的公式一致，只是用 `num_blocks × block_size` 表示可存储的 Token 总数。

## 链式 Block Hash 确认完整前缀

PagedAttention 通过链式 Block Hash 判断两个请求是否具有完全相同的前缀：

$$
\operatorname{hash}(\text{Block }n)=\operatorname{hash}(\operatorname{hash}(\text{Block }n-1),\ \operatorname{tokens}(n))
$$

第 $n$ 个块的哈希包含前一块的哈希和当前块的 Token，因此间接包含从开头到当前块的全部 Token 信息。对应位置的 Block Hash 相同，表示从请求开头到该块的前缀一致。

只有填满的 Block 才计算哈希。末块若只有三个 Token，后续 Decode 可能继续填充它，因此应在填满后再计算稳定哈希。

## 两张表连接前缀与物理块

`BlockHashToBlockMap` 是所有请求共享的全局表，将 `BlockHash` 映射到 `KVCacheBlock`。新请求按块计算哈希并查表：命中表示该前缀已被计算，可从对应 GPU 物理块直接读取 KV。

每个请求还保存一份 Block Table，对应代码中的 `req_to_blocks`。它按逻辑顺序记录当前请求使用的 `KVCacheBlock` 列表，将请求的逻辑块连接到分散的 GPU 物理块。

## 四个时刻中的分配、共享与驱逐

资料用 `block_size = 4` 的三个请求展示完整过程。示例用前缀首尾字符代表哈希，只是为了使链式前缀直观可读，不是真实哈希值。

### 时刻 0：第一个请求全部未命中

`request[0]` 包含 `ABCD / EFGH / IJKL` 三个逻辑块。它们的链式哈希用 `0xABCD`、`0xABGH` 和 `0xABKL` 表示。初始哈希表为空，三次查询全部 Miss，系统从 `free_block_queue` 分配物理 Block 0、1 和 2，得到 `req_to_blocks[0] = [0, 1, 2]`。三个块的 `ref_cnt` 都变为 1，并全部执行 Prefill。

### 时刻 1：第二个请求共享前两块

`request[1]` 的前八个 Token 同样是 `ABCD / EFGH`，最后是未填满的 `XYZ`。前两个哈希命中物理 Block 0 和 1，无需复制 KV，也无需重新执行这两块的 Prefill；系统只为 `XYZ` 分配物理 Block 3，得到 `req_to_blocks[1] = [0, 1, 3]`。物理 Block 0 和 1 的 `ref_cnt` 从 1 增加到 2，Block 3 为 1。

### 时刻 2 与 3：请求结束不等于缓存作废

`request[0]` 结束后，Block 0 和 1 仍被第二个请求使用，`ref_cnt` 从 2 降为 1；Block 2 的 `ref_cnt` 降为 0，回到 `free_block_queue`。但 Block 2 中的 KV 尚未被覆盖，`0xABKL → block_id:2` 的哈希映射也继续保留。

`request[1]` 结束后，Block 0、1 和 3 的 `ref_cnt` 全部降为 0 并返回空闲队列。返回队列只表示这些块当前无人使用、可以被重新分配；在被覆盖前，其中的 KV 和哈希记录仍然可以命中。

### 时刻 4：捡回空闲块并在覆盖时驱逐旧哈希

`request[2]` 包含 `ABCD / EFGH / PPPP / QQQQ`。前两块命中空闲队列中的 Block 0 和 1，系统将它们摘出队列并把 `ref_cnt` 设为 1。后两块 Miss，从队列分配 Block 3 和 2，得到 `req_to_blocks[2] = [0, 1, 3, 2]`。前两块跳过 Prefill，后两块重新计算。

Block 2 原本保存 `IJKL` 对应的 KV，现在要改存 `QQQQ`。只有到旧内容即将被覆盖的这一刻，`0xABKL → block_id:2` 才从哈希表中 Evict，并被新的 `0xABQQ` 替代。因此，资料中的驱逐不是定时清理，也不是请求结束后的缓存过期，而是物理块重新分配并真正覆盖旧 KV 时的元数据更新。

## 分页、映射、共享与回收

PagedAttention 把 KV Cache 切成固定大小、可分页、可映射且可共享的物理块。Block Table 恢复每个请求的逻辑顺序，Block Hash 让不同请求按完整前缀寻找相同物理块，`ref_cnt` 保证共享块不会在仍被使用时回收，`free_block_queue` 则让已结束请求的块在被覆盖前仍可被后续请求捡回复用。

本文讲解的是 vLLM 的 PagedAttention 与前缀缓存数据结构。开头引用了 Codex 抓包中 `cached_tokens` 按 512 递增的观察，但没有展示 OpenAI 服务端的物理 Block 大小或实现代码。
