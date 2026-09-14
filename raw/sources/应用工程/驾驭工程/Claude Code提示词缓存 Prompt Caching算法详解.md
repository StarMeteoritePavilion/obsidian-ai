---
title: Claude Code提示词缓存 Prompt Caching算法详解
source: https://www.bilibili.com/video/BV1FjRtBmEaH
author: 张司机在路上
created: 2026-05-07
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
---

# Claude Code提示词缓存 Prompt Caching算法详解

Claude Code 通过请求中的 `cache_control` 标记指定 Prompt Caching 的缓存边界。标记所在的 Block 是一个 Breakpoint：服务端把从请求开头到该 Block 的完整累积前缀写入缓存；后续请求读取缓存时，会从新的 Breakpoint 位置向前寻找此前写入的条目。

## 三个 `cache_control` 断点

一次 `hello` 请求的抓包中出现三个 `cache_control`：

1. 第一个位于 `system[1]` 的 Claude Code 身份提示末尾；
2. 第二个位于 `system[2]` 的行为准则末尾；
3. 第三个位于 `messages` 中最新用户输入 `hello` 的 Content Block 上。

本次标记使用 `type: ephemeral` 和 `ttl: 1h`。虽然 29 个工具定义本身没有携带 `cache_control`，但它们位于第一个断点之前，因此会随同此前的 System Block 一起进入该断点覆盖的缓存前缀。

## Automatic 与 Explicit

资料区分了两种 `cache_control` 用法。

**Automatic** 模式在请求顶层放置 `cache_control`，由服务器自动在最后一个 Block 设置一个 Breakpoint。它不要求客户端逐个选择边界位置。

**Explicit** 模式由客户端把 `cache_control` 直接挂到选定的 Block 上，最多可以设置 4 个 Breakpoint。Claude Code 使用 Explicit 模式，本次请求中的三个位置均由客户端选择。

三个断点并非都随对话移动。前两个固定在 System Prompt 中，第三个跟随最新用户消息向后移动：第一轮位于 `hello`，第二轮移到 `fine`，第三轮再移到 `thank you`。

## 两个固定锚点划分稳定层级

第一个 Breakpoint 覆盖 29 个工具定义、`system[0]` 和 Claude Code 身份提示 `system[1]`。资料把这段视为最稳定的前缀，并称同一 Claude Code 版本的用户可以复用它。

第二个 Breakpoint 在此基础上继续包含 `system[2]` 的行为准则。该 Block 含当前工作目录和系统信息，不同用户或项目可能不同；同一目录中的 Session 仍可以重复使用这段缓存。

第三个 Breakpoint 覆盖工具、全部 System Prompt 和当前对话历史。它持续移动，使每轮新增的 Assistant 回复和用户输入可以接在上一轮已经写入的对话前缀之后。

固定锚点分别保存全局较稳定部分和项目较稳定部分，移动锚点则追踪当前 Session 的增长。只在消息末尾保留一个断点，会失去前两层较短但更稳定的复用边界。

## 写缓存：每个断点对应一条累积前缀

缓存只在 Breakpoint 位置写入。资料用累积 Hash 描述三个缓存键：

```text
hash1 = hash(tools + system[0] + system[1])
hash2 = hash(tools + system[0] + system[1] + system[2])
hash3 = hash(tools + system[0] + system[1] + system[2] + user "hello")
```

一次请求因此会在三个位置写入三条前缀范围逐渐增长的缓存记录。这里的 Hash 表示资料用于解释匹配的逻辑模型；抓包能够确认断点位置和计量结果，不能直接观察服务端内部存储实现。

## 读缓存：从断点逐 Block 向前回溯

读取时，如果当前 Breakpoint 位置没有命中，服务端会向前一个 Block 继续查找以前写入的缓存条目，而不是仅判断当前哪些文字看起来稳定。

### 第一轮：`hello`

三个 Breakpoint 都是首次出现，没有已有条目，因此全部 Cache Miss。服务端计算三个累积前缀，并在三个断点分别写入缓存。

### 第二轮：`fine`

移动断点从 `hello` 到 `fine`。`fine` 对应的新累积前缀尚未写入，因此第一次查询 Miss；向前到 Assistant 对 `hello` 的回复仍然 Miss；继续向前到上一轮的 `hello`，命中已经写入的前缀。

从 `tools` 到 `hello` 的内容可以从缓存读取，`hello` 之后的 Assistant 回复和 `fine` 需要重新计算。完成后，服务端再在 `fine` 位置写入新的累积前缀。

### 第三轮：`thank you`

移动断点再次前移。`thank you` 和紧邻的 Assistant 回复均未写入；继续回溯到 `fine` 后，命中第二轮保存的条目。此前到 `fine` 的前缀从缓存读取，后续新增 Block 重新计算，并在 `thank you` 位置写入下一条记录。

日常短对话每轮通常只增加少量 Block，因此回溯两三步便能找到上一轮的用户消息缓存。

## 20 个 Block 的回溯上限

资料给出的第三条规则是：一次缓存查询最多向前检查 20 个 Block，仍未命中便停止回溯。

工具密集型任务可能在两次用户输入之间产生大量 `tool_use` 与 `tool_result`。如果一轮读取几十个文件、搜索网页并修改代码，可能新增二三十个 Block。下一轮用户发送“继续”时，移动 Breakpoint 从新消息向前回溯 20 个 Block，仍可能到不了上一轮写入缓存的 `thank you`。

这种情况下，移动断点无法复用上一轮保存的长对话前缀，相关内容需要重新计算。两个固定 System Breakpoint 仍是独立边界；20 Block 上限影响的是从当前断点寻找此前条目的范围，不能把它解释为服务器永久删除了原缓存。

## 缓存优化依赖前缀结构

官方简介还转述了 Claude Code 核心工程师 Thariq 的文章 *Lessons from building Claude Code: Prompt caching is everything*。其中的设计原则包括：把静态内容放在动态内容之前；把 Plan Mode 实现为两个工具，而不是切换整套工具集合；Tool Search 使用 `defer_loading` 发送轻量存根；Compact 复用原 Session 的 System、Tools 与 History 前缀。

对应的使用提醒是，不要在 Session 中途切换模型，也不要中途修改 MCP 或 Hook。模型缓存彼此隔离，而工具或 Hook 变化会改写靠前的请求前缀。

Prompt Caching 的关键不是只保存最新消息，而是选择多个稳定层级作为断点，并让移动断点沿对话增长。缓存能否命中，取决于此前是否在可回溯范围内写过相同累积前缀；断点位置、Block 数量和请求前部的稳定性共同决定复用范围。
