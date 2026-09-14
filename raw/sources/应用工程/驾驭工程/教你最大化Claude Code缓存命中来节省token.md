---
title: 教你最大化Claude Code缓存命中来节省token
source: https://www.bilibili.com/video/BV1ZQ5u6bEJ7
author: 张司机在路上
created: 2026-05-12
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
---

# 教你最大化Claude Code缓存命中来节省token

Claude Code 请求中的上下文按 `tools → system → CLAUDE.md／skills → messages` 排列。Anthropic 对完整前缀逐段计算 Hash，因此越靠左的内容越需要保持稳定：左侧变化会使其右侧前缀一起失去匹配，右侧变化的影响范围则相对较小。

影响缓存命中的四类动作，按照资料给出的影响范围从大到小，分别是中途切换模型、中途安装 MCP、修改 `CLAUDE.md` 或安装 Skill，以及请求间隔超过默认缓存有效期。

## 中途切换模型

Prompt Cache 保存的不是文字本身，而是 Transformer 各层 Attention 已经计算出的 KV Cache，即每一层的 Key 和 Value Tensor。Opus 与 Sonnet 的模型架构和权重不同，同一提示词产生的 KV 也不同，因此缓存按模型隔离。

如果主对话已使用 Opus 累积十万个 Token，再通过 `/model` 切换到 Sonnet，Sonnet 不能复用 Opus 的 KV Cache，需要重新计算并写入自己的缓存。资料采用的价格口径中，这些 Token 会从 `0.1×` 的缓存读取转为 `1.25×` 的五分钟缓存写入。

确实需要使用另一模型时，资料建议把任务隔离到 Subagent：主对话继续使用 Opus，另一个 Subagent 使用目标模型完成工作，最后只把交接结果返回主对话。资料还以 Explore Tool 和 WebSearch Tool 的 Agent 使用 Haiku 为例，说明不同模型可以通过独立上下文分工，而不必在同一 Session 中切换主模型。

## 中途安装 MCP

新安装的 MCP 会改变 `tools` 数组。工具层位于请求前缀最左侧，Hash 变化后，后面的 `system` 和 `messages` 也会失去原来的前缀匹配。

Claude Code 在启动时读取一次 MCP 配置，因此安装新 MCP 不会立即改变当前 Session 已加载的工具。真正触发变化的是随后使用 `/resume` 或 `/reload-plugins`：Claude Code 重新组装 `tools` 数组，新工具集合与此前缓存不再一致。

资料据此建议，在任务开始前一次性安装并配置所需 MCP，避免工作进行到一半才添加工具并重载环境。

## 修改 `CLAUDE.md` 或安装 Skill

`CLAUDE.md` 与 Skill 列表都属于 `messages` 层的注入上下文。资料展示的请求中，Skill 列表位于第二个 Block，`CLAUDE.md` 位于第三个 Block。

它们同样只在 Claude Code 启动时读取。当前 Session 中途修改 `CLAUDE.md` 或安装新 Skill，不会立刻进入当前上下文；使用 `/resume` 后，Claude Code 会重新组装 `messages`，变化位置之后的消息前缀需要重新建立缓存。

开始复杂任务之前，应先确定需要的 Skill，并把任务所需的重要项目规则一次性写入 `CLAUDE.md`。资料不建议为了让中途变更生效而在同一任务中使用 `/resume`。

## 请求间隔超过五分钟

资料称，官方默认 Prompt Cache TTL 为五分钟。超过五分钟没有新请求时，服务器会主动删除缓存条目。这与前缀 Hash 不匹配不同：即使下一次发送完全相同的请求，已经过期的上下文仍需重新计算并写入缓存。

长任务可能因为人工检查、规划下一步或 Claude Code 自身处理而超过五分钟。作者因此建议复杂任务启用一小时 TTL，在启动 Claude Code 前设置：

```bash
export ENABLE_PROMPT_CACHING_1H=1
```

资料采用的价格口径中，五分钟缓存写入为普通输入价格的 `1.25×`，一小时缓存写入为 `2×`。一小时写入成本更高，但在复杂任务中可以降低五分钟过期后整段重新写入的风险。是否更划算仍取决于任务间隔、前缀长度和实际命中情况。

## 稳定前缀的实践原则

四类问题对应同一原则：任务开始前确定模型，配置所需 MCP、Skill 与 `CLAUDE.md`，并根据任务时长选择 TTL；Session 进行中尽量保持前缀左侧内容稳定。

这些措施不会减少当前任务真正新增的消息，也不能保证每次请求都命中缓存。它们只是避免因模型、工具集合、注入上下文或缓存过期而让已经计算的稳定前缀失去复用条件。
