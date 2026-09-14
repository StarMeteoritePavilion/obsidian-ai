---
title: Claude Code：模型、工具、注入与 TTL 的缓存命中边界
source: https://www.bilibili.com/video/BV1ZQ5u6bEJ7
author: 张司机在路上
published: 2026-05-12
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - Prompt Caching
  - KV Cache
  - Token
  - 上下文工程
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：模型、工具、注入与 TTL 的缓存命中边界

原始资料：[[raw/sources/应用工程/驾驭工程/教你最大化Claude Code缓存命中来节省token|教你最大化Claude Code缓存命中来节省token]]

## 核心结论

资料把 Claude Code 请求前缀表示为 `tools → system → CLAUDE.md／skills → messages`，并按位置从左到右列出四类缓存失效因素：切换模型、安装 MCP 后重载、修改 `CLAUDE.md` 或安装 Skill 后恢复 Session，以及超过默认五分钟 TTL。越靠左的变化，后续失去匹配的范围越大。

## 四类失效因素

1. **模型变化**：Prompt Cache 复用的是模型计算出的 Key／Value Tensor，不同模型的架构和权重不同。资料以 Opus 累积十万个 Token 后切换 Sonnet 为例，说明 Sonnet 不能读取 Opus 的缓存；建议需要其他模型时使用独立 Subagent，再把结果交回主对话。
2. **工具集合变化**：安装 MCP 本身不会立即改变当前 Session；使用 `/resume` 或 `/reload-plugins` 后，Claude Code 重新组装 `tools` 数组，新增工具会改变左侧前缀。
3. **注入上下文变化**：资料所示 Skill 列表位于 `messages` 第二个 Block，`CLAUDE.md` 位于第三个 Block。二者在启动时读取，修改后通过 `/resume` 重建 `messages` 才会进入请求，同时使变化位置之后的缓存失去匹配。
4. **TTL 过期**：超过五分钟没有新请求时，资料称服务端会删除默认缓存条目；下一次请求即使内容相同也需要重新写入。作者建议复杂任务在启动前设置 `export ENABLE_PROMPT_CACHING_1H=1`，把 TTL 延长到一小时。

资料采用的价格口径为：缓存读取 `0.1×`，五分钟缓存写入 `1.25×`，一小时缓存写入 `2×`。一小时 TTL 是否节省费用取决于请求间隔和可复用前缀长度，不能把作者对复杂任务的建议外推为所有会话的固定最优选择。

## 与后续 ToolSearch 资料的边界

本资料讨论的是安装新 MCP 后，Claude Code 在恢复或重载时改变启动工具集合。后续 ToolSearch 资料讨论的是从启动时已经存在的 Deferred Tool 目录按需加载定义：在支持 `defer_loading` 与 `tool_reference` 的协议中，新定义不参与原缓存前缀 Hash，引用追加在历史末尾。两者处理的对象和时间点不同，不能把“按需加载可保持缓存”解释为“中途修改 MCP 配置永远不会影响缓存”。

## 第三方代理中的 Attribution Header

后续抓包补充了第五种来源特定的失效入口：Claude Code 2.1.119 在 System Prompt 第一块加入 `x-anthropic-billing-header`，其中 `cch` 在同一 Session 的三轮请求中依次为 `97bd6`、`24c2d`、`ead88`。这段内容位于三个缓存断点之前；第三方服务若把完整 System Prompt 纳入前缀 Hash，三个断点都会失去匹配。它与本页四类因素不同：不是用户中途修改配置，而是客户端主动改变最左侧系统文本。是否受影响取决于代理实现，必须抓包确认。（[[wiki/sources/Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header|第三方 API 的 cch 缓存失效]]）

## 证据边界

- 四类因素的伤害排序来自资料按请求位置给出的教学判断；实际损失还取决于变化位置、前缀长度、缓存状态和后续请求数量。
- `ENABLE_PROMPT_CACHING_1H=1`、五分钟默认 TTL 与价格倍率属于视频所示 Claude Code 和 Anthropic 产品口径，不能无条件外推到其他版本、模型或接入方式。
- Explore Tool、WebSearch Tool 使用 Haiku 是资料对当时 Claude Code 行为的描述，不代表这些 Agent 始终固定使用同一模型。
- 本资料说明客户端请求怎样变化以及计量如何受影响，不直接证明 Anthropic 服务端的完整 Hash、存储和淘汰实现。

## 关联

- 多轮缓存计量：[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本]]
- 第三方 API 的 cch 失效：[[wiki/sources/Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header]]
- 缓存内容与层级：[[wiki/sources/模型推理优化：KV Cache 与 Prompt Cache 的复用层级]]
- 延迟工具加载：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- Claude Code 请求结构：[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- 通用成本边界：[[wiki/sources/模型推理优化：Token 成本、KV Cache 与缓存机制]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
