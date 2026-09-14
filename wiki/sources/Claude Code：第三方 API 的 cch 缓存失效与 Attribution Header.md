---
title: Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header
source: https://www.bilibili.com/video/BV1m2LG6WEdH
author: 张司机在路上
published: 2026-05-16
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - Prompt Caching
  - Attribution Header
  - CCH
  - 第三方 API
  - Bun
  - Zig
  - 上下文工程
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：第三方 API 的 cch 缓存失效与 Attribution Header

原始资料：[[raw/sources/应用工程/驾驭工程/如何修复Claude Code给第三方大模型用户挖的坑|如何修复Claude Code给第三方大模型用户挖的坑]]

## 核心结论

Claude Code 从 2.1.36 开始把 `x-anthropic-billing-header` 作为 System Prompt 第一块发送，其中五位十六进制 `cch` 每轮变化。Anthropic 自有服务可能会识别并特殊处理它；第三方 Anthropic 兼容代理、Bedrock 或本地 vLLM 若把完整 System Prompt 纳入前缀 Hash，三个缓存断点都可能随 `cch` 变化而 Miss。对已经确认受影响的第三方接入，可通过 `CLAUDE_CODE_ATTRIBUTION_HEADER=0` 移除这段内容。

## 三轮抓包

作者用 `claude-tap` 对同一 Session 中的 `Hello`、`Fine`、`Thank you` 三轮请求进行比较。首个 System Block 包含 `cc_version`、`cc_entrypoint` 和 `cch`：

- `cc_version=2.1.119.af2`：版本号为 2.1.119，`af2` 为三位十六进制完整性指纹；
- `cc_entrypoint=cli`：普通 CLI 入口；`-p` 模式使用 `sdk-cli`；
- `cch`：三轮依次为 `97bd6`、`24c2d`、`ead88`。

`x-anthropic-billing-header` 是 System Prompt 文本，不是 HTTP Header。它位于三个 `cache_control` 断点之前，因此任一字符变化都会改变三个断点之前的完整前缀。第一轮写入的缓存可能在后两轮全部失去匹配。

## 生成链路

源码快照中的 TypeScript 入口位于 `src/constants/system.ts`：

- `getAttributionHeader(fingerprint: string)` 负责组装 Attribution Header；
- `isAttributionHeaderEnabled()` 决定是否返回空字符串；
- `NATIVE_CLIENT_ATTESTATION` 启用时，JavaScript 层先写入 `cch=00000`；
- Bun Native HTTP Stack 在请求发送前原位覆盖占位符；
- 源码注释将原生实现指向 `bun-anthropic/src/http/Attestation.zig`。

五个零与真实 `cch` 等长，因此替换时不用改变 `Content-Length` 或重新分配 Buffer。计算发生在 JavaScript 引擎之外，仅拦截 `fetch` 或 Monkey Patch JavaScript HTTP 层无法取得最终值。

## 用途与适用边界

作者将 `cch` 解释为 Claude Code 客户端 Attestation：OAuth Token 与 Pro／Max 订阅绑定，服务端需要区分真实 Claude Code 与直接复用订阅 Token 的其他程序。视频同时展示 `sub2api` 已经复现相关算法，说明该逻辑并非不可分析。

“Anthropic 服务端在缓存计算时跳过这段内容”是作者根据官方服务不受影响作出的推测，抓包和源码快照没有直接证明服务端缓存键算法。资料也没有证明所有第三方服务必然失效；是否受影响取决于代理怎样处理 System Prompt 与 Prompt Cache。

## 处理办法

只在第三方 API 环境中，先用连续请求抓包确认 `cch` 每轮变化并与缓存 Miss 对应，再在 `~/.claude/settings.json` 中设置：

```json
{
  "env": {
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0"
  }
}
```

重启后复测。作者的结果是 Billing Header 消失，System Block 从三个变为两个，稳定前缀不再被 `cch` 改写。这项设置只处理 Attribution Header 导致的缓存失效，不是推理变慢或 Token 增长的通用修复，也不应从第三方接入直接外推到 Anthropic 自有服务或订阅 OAuth 路径。

## 证据边界

- 版本 2.1.119、三组 `cch`、三个缓存断点和关闭后的两个 System Block 来自作者当次抓包，不能视为所有 Claude Code 版本的固定结构。
- `src/constants/system.ts`、函数名、环境变量、`NATIVE_CLIENT_ATTESTATION` 与 `Attestation.zig` 路径由源码画面确认；视频没有提供可复核的 Commit，因此只代表所展示快照。
- `cch` 的防复用用途来自源码注释与作者解释；第三方缓存失效仍需在实际代理上抓包验证。
- `CLAUDE_CODE_ATTRIBUTION_HEADER=0` 的适用范围限于不需要官方订阅 Attestation 的第三方 API 接入，不能无条件用于所有 Claude Code 配置。

## 关联

- 多轮缓存计量：[[wiki/sources/Claude Code：多轮对话的前缀缓存与 Token 成本]]
- 缓存命中边界：[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- 请求结构：[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- ToolSearch 与代理协议：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
