---
title: 解密多轮对话Claude Code如何计算token消耗
source: https://www.bilibili.com/video/BV1KGoyBGEjN
author: 张司机在路上
created: 2026-04-27
tags:
  - AI
  - Claude Code
  - Anthropic
  - Prompt Caching
  - Token
  - 上下文工程
  - 驾驭工程
  - 应用工程
---

# 解密多轮对话Claude Code如何计算token消耗

一次只输入 `hello` 的 Claude Code 请求可能处理数万个 Token，但连续对话十轮并不等于把首轮成本简单乘以十。Claude Code 每轮仍会发送完整请求，Anthropic 的 Prompt Caching 则会复用保持不变的前缀，只计算新增的对话内容。

## 用 claude-tap 对比相邻请求

此前使用的 `claude-trace` 通过修改 Claude Code 代码记录请求。作者使用 NPM 安装的 Claude Code 2.1.112 时可以运行该工具，但在 2.1.119 中已经无法继续使用；当时官方也已不再支持 NPM 安装。

作者改用 `claude-tap`。它在本地启动代理服务器和 Claude Code，使 API 流量经过代理转发，并在会话退出后生成 HTML 记录。页面既保留完整 JSON，也把 `tools`、`system`、`messages`、响应和 SSE 事件拆成独立模块。相邻请求的对比视图会高亮新增内容，把未变化内容显示为灰色。

收集请求时运行：

```bash
claude-tap --tap-live
```

测试依次发送 `Hello`、`Fine` 和 `Thank you`，形成三次 API 请求。

## 前缀缓存复用什么

每次请求都会按固定顺序包含工具定义、系统提示词、用户注入的上下文和对话历史。这些内容共同构成缓存查询所使用的前缀，当前用户输入位于末尾。

Prompt Caching 的正式概念是 Prefix Caching，即前缀缓存。服务端对前缀计算 Hash；两次请求的前缀相同时，后一轮可以直接读取已经计算的缓存，变化位置之后的内容则需要重新计算。资料把匹配要求概括为字符必须保持一致：前部内容越稳定，后续对话越容易持续命中。

本次首轮请求包含 31 个工具定义、Anthropic 编写的三段系统提示词，以及五个消息 Block。前四个 Block 是 Claude Code 注入的 Hook 配置、MCP 指南、Skill 列表和 `CLAUDE.md` 等项目上下文，最后一个才是 `Hello`。因此，用户输入之前的大段稳定内容可以作为可复用前缀。

## 第一轮：冷启动并写入缓存

第一轮响应的 `usage` 记录为：

- `input_tokens`：6；
- `cache_creation_input_tokens`：48,654；
- `cache_read_input_tokens`：0。

6 个输入 Token 对应 `Hello` 及其格式；此前没有可读取的同一前缀缓存，因此 48,654 个 Token 首次写入缓存。这是本次三轮对话的冷启动。

## 第二轮：读取首轮前缀

第二次请求的消息从一条增加到三条：原来的 `Hello`、Assistant 回复，以及新的用户消息 `Fine`。系统提示词和工具定义没有变化。

对应 `usage` 为：

- `input_tokens`：6；
- `cache_creation_input_tokens`：24；
- `cache_read_input_tokens`：48,654。

`cache_read_input_tokens` 与上一轮的 `cache_creation_input_tokens` 完全相同，表示首轮写入的 48,654 Token 前缀全部命中。新增的 Assistant 回复和 `Fine` 共形成 24 个新的缓存写入 Token。

## 第三轮：缓存随历史增长

第三次请求又增加 Assistant 回复和用户消息 `Thank you`，消息总数从三条变为五条，其他前缀仍然保持不变。对应 `usage` 为：

- `input_tokens`：6；
- `cache_creation_input_tokens`：29；
- `cache_read_input_tokens`：48,678。

第三轮读取的 48,678 Token，正好等于第二轮读取的 48,654 Token 加上第二轮新写入的 24 Token：

$$
48{,}654+24=48{,}678
$$

这三轮数据呈现出同一关系：后一轮的缓存读取量由上一轮已经读取的前缀与上一轮新增写入共同组成。对话历史从一条消息增长到五条消息时，两轮新增缓存共 53 Token，而数万 Token 的工具定义、系统提示词和用户注入上下文不必重复计算。

## 缓存读写的价格差异

资料展示的价格倍率为：普通输入按 `1×` 计费，5 分钟缓存写入按 `1.25×`，1 小时缓存写入按 `2×`，缓存读取按 `0.1×`。这些倍率是视频发布时采用的 Anthropic 价格口径。

按 1 小时缓存写入倍率计算，三轮输入的折算成本分别为：

- 第一轮：$48{,}654\times2+6=97{,}314$；
- 第二轮：$24\times2+48{,}654\times0.1+6\approx4{,}919$；
- 第三轮：$29\times2+48{,}678\times0.1+6\approx4{,}932$。

首轮冷启动需要付出缓存写入成本；从第二轮开始，稳定前缀主要按缓存读取价格计费，每轮只为新增的少量历史写入缓存。对话越长，可复用前缀在总输入中的占比通常越高，缓存相对于每轮全部重新计算的优势也越明显。

Prompt Caching 并没有让完整请求从网络传输和 Token 计量中消失。它复用的是稳定前缀已经完成的计算：工具定义、系统提示词、项目上下文和历史消息仍随请求携带，服务端通过缓存读取降低重复计算与对应费用，而新增加的对话内容继续产生新的输入和缓存写入。
