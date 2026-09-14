---
title: Claude Code：多轮对话的前缀缓存与 Token 成本
source: https://www.bilibili.com/video/BV1KGoyBGEjN
author: 张司机在路上
published: 2026-04-27
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - Prompt Caching
  - Token
  - 上下文工程
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：多轮对话的前缀缓存与 Token 成本

原始资料：[[raw/sources/应用工程/驾驭工程/解密多轮对话Claude Code如何计算token消耗|解密多轮对话Claude Code如何计算token消耗]]

## 核心结论

Claude Code 每轮仍发送工具定义、系统提示词、项目上下文和完整对话历史，但 Anthropic Prompt Caching 会复用保持不变的前缀。本次三轮抓包中，缓存读取量从 0 增至 48,654、48,678 Token，后两轮只新增写入 24、29 Token；因此，多轮成本不能用首轮冷启动成本直接乘以轮数。

## 三轮抓包

作者用 `claude-tap --tap-live` 依次发送 `Hello`、`Fine`、`Thank you`，记录到以下输入计量：

| 请求 | `input_tokens` | `cache_creation_input_tokens` | `cache_read_input_tokens` |
| --- | ---: | ---: | ---: |
| 第一轮 | 6 | 48,654 | 0 |
| 第二轮 | 6 | 24 | 48,654 |
| 第三轮 | 6 | 29 | 48,678 |

第三轮缓存读取满足 $48{,}678=48{,}654+24$。稳定前缀沿对话历史滚动增长，后续请求只需为新增的 Assistant 回复与用户消息计算并写入缓存。三轮消息数量从一条增至五条，两轮新增缓存合计 53 Token。

## 前缀结构与计费

本次请求把 31 个工具定义、三段 System Prompt、Hook、MCP 指南、Skill 列表、`CLAUDE.md` 和对话历史按固定顺序放在用户当前输入之前。资料将缓存匹配概括为前缀 Hash：前部内容和顺序稳定时可以复用，变化位置之后需要重新计算。

视频发布时展示的倍率为普通输入 `1×`、5 分钟缓存写入 `1.25×`、1 小时缓存写入 `2×`、缓存读取 `0.1×`。按 1 小时写入计算，三轮折算成本为 97,314、约 4,919、约 4,932 个普通输入 Token 价格单位。该价格只属于资料采用的时间与产品口径，不是跨模型、跨版本的固定费率。

## 证据边界

- 48,654 Token 与 31 个工具属于作者当次 Claude Code 配置和抓包，不代表 Claude Code 的固定启动成本；前一资料的 `hello` 抓包合计 30,986 个输入 Token，二者不能直接互换。
- 前一抓包记录 10 个完整工具定义和四个 System Text Block，本次资料称有 31 个工具定义和三段 System Prompt。两次抓包的版本、配置与计数口径不同，只能分别描述各自请求，不能合并为 Claude Code 的固定结构数量。
- 缓存读取降低的是稳定前缀的重复计算和对应费用，不表示完整请求、历史 Token 或网络传输消失。
- 资料把匹配要求口语化为“一个字符不差”。本次抓包支持稳定前缀完全命中及新增后缀继续写入，不能单独证明 Anthropic 服务端的全部缓存实现细节。
- `claude-tap` 通过本地代理观察客户端请求和服务端计量；它能证明抓包字段怎样变化，不能直接观察服务端内部存储结构。

## 关联

- 上一篇：[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- 下一篇：[[wiki/sources/Claude Code：tool_use、tool_result 与客户端工具闭环]]
- 缓存命中实践：[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- 缓存断点与回溯：[[wiki/sources/Claude Code：cache_control 断点与 20 Block 前缀回溯]]
- 第三方 API 的 `cch` 失效：[[wiki/sources/Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header]]
- 缓存内容与层级：[[wiki/sources/模型推理优化：KV Cache 与 Prompt Cache 的复用层级]]
- 工具延迟加载：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- Codex 前缀缓存：[[wiki/sources/模型推理优化：Codex 自动前缀缓存]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
