---
title: Claude Code：cache_control 断点与 20 Block 前缀回溯
source: https://www.bilibili.com/video/BV1FjRtBmEaH
author: 张司机在路上
published: 2026-05-07
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - Prompt Caching
  - cache_control
  - Prefix Caching
  - 上下文工程
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：cache_control 断点与 20 Block 前缀回溯

原始资料：[[raw/sources/应用工程/驾驭工程/Claude Code提示词缓存 Prompt Caching算法详解|Claude Code提示词缓存 Prompt Caching算法详解]]

## 核心结论

资料中的 Claude Code 使用 Explicit `cache_control`，在 `system[1]`、`system[2]` 和最新用户消息上设置三个 Breakpoint。每个断点写入从请求开头到当前位置的累积前缀；读取时若当前位置 Miss，最多向前回溯 20 个 Block，寻找此前实际写入的条目。

## 三层断点

本次抓包的第一个断点覆盖 29 个工具、`system[0]` 和身份提示，第二个继续覆盖行为准则，第三个覆盖到当前用户消息。前两个保持不动，第三个依次从 `hello` 移到 `fine`、`thank you`。

Automatic 模式由服务器自动在最后一个 Block 设置一个断点；Explicit 模式由客户端选定 Block，最多设置 4 个。三个断点的 `cache_control` 均显示 `type: ephemeral`、`ttl: 1h`。

## 三轮读写

第一轮三个位置均 Miss，并分别写入三条累积前缀。第二轮从 `fine` 向前检查，在 Assistant 回复处仍 Miss，到 `hello` 命中；第三轮从 `thank you` 回溯到 `fine` 命中。命中点之前从缓存读取，之后的新增 Block 重新计算，再在最新用户消息处写入新条目。

工具密集的一轮可能加入二三十个 `tool_use`／`tool_result` Block。下一轮若在 20 Block 内找不到上一条缓存，移动断点便不能复用那段长对话前缀。该上限描述查询范围，不表示既有缓存被删除，也不否定两个固定 System 断点可以独立命中。

## 跨资料边界

本资料图中显示 29 个工具，较早的多轮计量资料记录 31 个工具；两次抓包的版本、配置或计数口径没有被证明相同，不能合并为固定数量。

资料称第一个固定断点可由同版本 Claude Code 用户共享。后续第三方 API 抓包却显示 Claude Code 2.1.119 的 `system[0]` 含每轮变化的 `cch`，可能使不识别该约定的代理在三个断点全部 Miss。两者适用的服务端协议和接入路径不同，因此“同版本即可共享”不能外推到任意第三方代理。

本资料解释缓存条目在哪里写、怎样查找；后续 KV Cache 专题解释命中后复用的是 Prefill 产生的 K／V 中间状态。断点、累积前缀与 20 Block 回溯属于作者依据当次抓包和官方文档给出的协议说明，不构成 Anthropic 服务端物理 Hash、存储或淘汰实现的完整证明。

## 关联

- 前置计量：[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本]]
- 合集上一篇：[[wiki/sources/Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离]]
- 合集下一篇：[[wiki/sources/Claude Code：Thinking 模式、Adaptive 与 Effort]]
- 缓存命中边界：[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- 第三方 `cch` 失效：[[wiki/sources/Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header]]
- 缓存内容与层级：[[wiki/sources/模型推理优化：KV Cache 与 Prompt Cache 的复用层级]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
