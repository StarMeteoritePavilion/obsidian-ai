---
title: Claude Code：tool_use、tool_result 与客户端工具闭环
source: https://www.bilibili.com/video/BV1sJ9tBQEmr
author: 张司机在路上
published: 2026-04-30
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - Opus
  - Tool Use
  - Bash
  - SSE
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：tool_use、tool_result 与客户端工具闭环

原始资料：[[raw/sources/应用工程/驾驭工程/揭秘Opus模型如何指挥Claude Code调用工具|揭秘Opus模型如何指挥Claude Code调用工具]]

## 核心结论

Claude Code 的客户端工具调用至少跨越两次模型请求：Opus 先返回指定工具与参数的 `tool_use`，Claude Code 在本地执行命令并以相同 ID 返回 `tool_result`，Opus 再读取结果、生成自然语言总结并以 `end_turn` 结束。模型决定和解释，客户端执行。

## Bash 抓包链路

作者让 Opus 4.7 回答“git有什么改动”。第一次响应中的工具 Block 使用 `type: tool_use`、以 `toolu_01` 开头的唯一 `id`、`name: Bash`，参数经 `input_json_delta` 流式组成：

```json
{
  "command": "git status && echo \"\\n---DIFF---\" && git diff",
  "description": "Show git status and diff"
}
```

响应末尾的 `stop_reason: tool_use` 表示模型等待客户端返回结果。Claude Code 执行命令后，在第二次请求的 `user` 消息中加入 `type: tool_result`；其 `tool_use_id` 与上一轮 `tool_use.id` 完全一致，`content` 保存 `git status` 和 `git diff` 的原始输出。

第二次响应只有普通 `text` Block，概括 `main` 分支中 `README.md`、`hello.py` 的修改以及 `.traces/`、`notes.md` 等未跟踪内容，最终 `stop_reason` 为 `end_turn`。抓包因此清楚区分了模型生成结构化行动意图、客户端改变外部环境、模型解释结果三个阶段。

## 执行位置边界

- Bash、Edit 等 Client-executed Tool 在用户电脑上执行；模型只能读取客户端返回的结果，不能直接观察终端执行过程。
- WebSearch、WebFetch 等 Server-executed Tool 由 Anthropic 服务器执行，客户端接收服务端结果。后续 WebSearch 抓包进一步显示，服务器端工具还可能由独立 Haiku 子 Agent 处理，再把压缩结果交给 Opus 主 Agent。

两类工具可以使用相似的 `tool_use` 决策入口，但执行位置、可见数据、权限和返回链路不同，不能仅凭“模型调用了工具”判断动作发生在哪里。

## 证据边界

- Opus 4.7、具体 Git 命令、文件状态和字段顺序来自作者当次 Claude Code 2.1.119 与 `claude-tap` 抓包，不代表所有版本的固定请求。
- 资料用“握手”解释 `tool_use → tool_result → end_turn`，这是流程类比，不表示该机制等同于 HTTP 握手协议。
- `tool_use_id` 证明结果与调用如何配对，不证明命令执行正确；结果仍需要由客户端退出状态、文件状态或独立验证确认。
- 客户端工具由 Claude Code 代模型执行，不表示默认具备无限权限；实际能力仍受当时工具定义、权限配置和运行环境约束。

## 关联

- 上一篇：[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本]]
- 下一篇：[[wiki/sources/Claude Code：WebSearch 子 Agent、服务端搜索与攻击面隔离]]
- 请求结构：[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- 工具延迟加载：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- Agent 最小链路：[[wiki/syntheses/AI Agent：从工具调用到可信行动]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
