---
title: 如何修复Codex总是重新连接Reconnecting
source: https://www.bilibili.com/video/BV1ASjx6XEcu
author: 张司机在路上
created: 2026-06-21
tags:
  - AI
  - Codex
  - 驾驭工程
  - 应用工程
---

# 如何修复Codex总是重新连接Reconnecting

Codex 在回答前可能连续显示 `Reconnecting`，甚至重连五次后才恢复正常。该现象不一定意味着模型响应慢或服务器繁忙：普通 HTTPS 请求能够通过，不代表以 `wss://` 开头的 WebSocket 连接在代理、VPN 或防火墙环境中同样稳定。

对于这类 WebSocket 连接不稳定、HTTPS 仍然可用的情况，可以通过用户级 Codex 配置关闭 WebSocket，让请求直接走 HTTPS Streaming。

## Responses API 的两种传输方式

Codex 会通过 Responses API 把上下文发往 OpenAI 后端。同一套 API 可以使用两种传输方式。

HTTPS Streaming 会为每次请求发送 HTTP POST，再由服务器通过 Server-Sent Events（SSE）逐步返回响应。WebSocket 则建立一条长连接，客户端与服务器可以在同一连接上持续交换数据，适合 Agent 多轮、流式且交互频繁的场景。

OpenAI 在 2026 年 4 月 22 日发布的工程文章 *Speeding up agentic workflows with WebSockets in the Responses API* 介绍了这种 WebSocket 传输方式。

## Codex 如何选择传输路径

相关逻辑位于 `codex-rs` 的 `client.rs`。`stream()` 函数先读取当前模型提供方配置中的 `wire_api`：当其值为 `responses` 时，函数再调用 `responses_websocket_enabled()` 判断是否使用 WebSocket。

`responses_websocket_enabled()` 会检查两个条件：

1. 当前模型提供方的 `supports_websockets` 是否为 `false`。
2. 当前会话的 `disable_websockets` 标志是否已经启用。

任一条件成立时，函数都会返回 `false`，请求随即进入 HTTPS Streaming 路径。视频中的代码说明，WebSocket 连续重试失败后，Codex 也会把当前会话的 `disable_websockets` 标志设为启用；这解释了为什么连续重连后仍能恢复正常回答。

## 通过配置直接使用 HTTPS

在 `~/.codex/config.toml` 中将当前模型提供方改为专门的 HTTP 配置：

```toml
model_provider = "openai_http"

[model_providers.openai_http]
name = "OpenAI HTTP"
wire_api = "responses"
requires_openai_auth = true
supports_websockets = false
```

这段配置分别承担以下作用：

- `model_provider = "openai_http"` 选择新配置的模型提供方。
- `[model_providers.openai_http]` 定义该模型提供方的信息。
- `wire_api = "responses"` 保持使用 Responses API。
- `requires_openai_auth = true` 保持 OpenAI 登录验证。
- `supports_websockets = false` 让 `responses_websocket_enabled()` 直接返回 `false`，从而进入 HTTPS Streaming 路径。

解决方法的关键不是更换模型或 API，而是关闭 WebSocket 传输。该配置针对 HTTPS 可用、WebSocket 链路不稳定的情况；它并不把所有 `Reconnecting` 现象都归因于 WebSocket。

## 为什么作者选择 HTTPS

作者为制作“Codex工作原理”系列而分析请求上下文。按照作者的观察，WebSocket 方式会维护会话状态，后续对话只传输新增的上下文，不便于观察完整历史；HTTPS 方式更容易逐次检查上下文变化。

作者还指出，当时国内大模型厂商适配 Codex 时通常提供 HTTPS API，部分实现只兼容 Chat Completions API，尚未兼容 Responses API，更没有提供 WebSocket 传输。因此，HTTPS 在这些兼容场景中更普遍，也更便于分析。尽管 WebSocket 的实时性更好，作者个人使用时没有感受到明显差异。
