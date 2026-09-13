---
title: Codex：禁用 WebSocket 解决重复重连
source: https://www.bilibili.com/video/BV1ASjx6XEcu
author: 张司机在路上
published: 2026-06-21
ingested: 2026-09-13
updated: 2026-09-13
tags:
  - AI
  - Codex
  - WebSocket
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Codex：禁用 WebSocket 解决重复重连

原始资料：[[raw/sources/应用工程/驾驭工程/如何修复Codex总是重新连接Reconnecting|如何修复Codex总是重新连接Reconnecting]]

## 核心结论

当普通 HTTPS 请求可用、WebSocket 链路却受代理、VPN 或防火墙影响时，Codex 可能在回答前连续显示 `Reconnecting`。本资料给出的处理方法是在用户级配置中保留 Responses API 与 OpenAI 身份验证，仅把模型提供方的 `supports_websockets` 设为 `false`，让传输直接进入 HTTPS Streaming 路径。

这是一项传输能力配置，不是模型或 API 替换。它只适用于确认 HTTPS 可用而 WebSocket 不稳定的情形，不能解释所有重连故障。

## 路径选择机制

资料展示的 `codex-rs/client.rs` 代码先由 `stream()` 检查 `wire_api`，再由 `responses_websocket_enabled()` 读取模型提供方的 `supports_websockets` 和会话级 `disable_websockets`。模型提供方声明不支持 WebSocket，或当前会话已在重试失败后关闭 WebSocket，都会使请求回退到 HTTPS Streaming。

配置保持 `wire_api = "responses"` 和 `requires_openai_auth = true`，真正改变传输分支的是 `supports_websockets = false`：

```toml
model_provider = "openai_http"

[model_providers.openai_http]
name = "OpenAI HTTP"
wire_api = "responses"
requires_openai_auth = true
supports_websockets = false
```

## 使用边界

- 配置与源码行为对应视频发布时展示的 Codex 版本，其他版本应按实际配置定义和源码重新核对。
- 作者为观察完整请求上下文而偏好 HTTPS，并认为个人使用时与 WebSocket 的体感差异不大；这是作者的使用经验，不是跨网络环境的性能结论。
- 资料称部分国内模型服务当时只兼容 HTTPS 或 Chat Completions API；该判断有时间和厂商范围，不应外推为所有兼容服务的固定能力。

## 关联

- [[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
- [[wiki/sources/上下文工程：提示词、上下文与 Harness 的职责边界]]
