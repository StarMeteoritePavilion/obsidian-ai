---
title: 如何修复Claude Code给第三方大模型用户挖的坑
source: https://www.bilibili.com/video/BV1m2LG6WEdH
author: 张司机在路上
created: 2026-05-16
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
---

# 如何修复Claude Code给第三方大模型用户挖的坑

Claude Code 配置第三方 API 后，如果出现推理变慢、Token 消耗暴涨，问题不一定来自模型服务本身。Claude Code 从 2.1.36 开始，会在每个 API 请求的 System Prompt 第一块加入一行 `x-anthropic-billing-header`；其中的五位十六进制 `cch` 每次请求都会变化。第三方服务若把整段 System Prompt 原样纳入前缀 Hash，变化的 `cch` 就可能使缓存持续失效。

这个问题只针对使用第三方 Anthropic 兼容代理、Bedrock 或本地 vLLM 等接入方式的场景。作者认为 Anthropic 自有服务会识别并特殊处理这段内容，但抓包无法直接观察服务端如何计算缓存键，因此这一点保留为作者推测。

## 每轮都会变化的 Billing Header

作者用 `claude-tap` 对同一 Session 中的三次请求进行比较，用户依次输入 `Hello`、`Fine` 和 `Thank you`。三轮请求的首个 System Block 都包含类似内容：

```text
x-anthropic-billing-header: cc_version=2.1.119.af2; cc_entrypoint=cli; cch=97bd6;
```

其中三个字段分别表示：

- `cc_version`：`2.1.119` 是 Claude Code 版本号，后面的 `af2` 是三位十六进制完整性指纹。
- `cc_entrypoint`：记录客户端入口；普通命令行模式为 `cli`，使用 `-p` 参数启动时变为 `sdk-cli`。
- `cch`：五位十六进制字符串，在本次三轮请求中依次为 `97bd6`、`24c2d` 和 `ead88`。

这里的 `x-anthropic-billing-header` 不是 HTTP Header，而是请求上下文中的 System Prompt 文本，并且排在系统提示词最前面。正是这个位置使它可能影响后续全部缓存断点。

## `cch` 为什么会让第三方缓存失效

资料将 Claude Code 的请求前缀概括为 `tools → system → messages`。服务端根据 `cache_control` 标记建立缓存断点：从请求开头到对应 Block 末尾的全部内容共同参与前缀匹配。

本次请求有三个缓存断点：

1. System Prompt 第二个 Block 末尾；
2. System Prompt 第三个 Block 末尾；
3. User Message 所在位置。

`cch` 位于 System Prompt 第一个 Block，在所有三个断点之前。它只要发生变化，三个断点对应的完整前缀就都会变化。第一轮刚写入的缓存，第二轮和第三轮便可能因为 `cch` 不同而无法匹配。

作者推测，Anthropic 自有服务知道这段 System Prompt 是 Claude Code 注入的归因信息，因此在计算缓存键时会跳过或特殊处理它。普通第三方转发服务并不知道这一内部约定，可能把完整 `system` 数组直接用于 Hash，导致每轮缓存都 Miss。作者称，从 2026 年 2 月起已有多个 GitHub Issue 报告 `cch` 引发的缓存失效，但没有观察到 Anthropic 对这些报告作出回应。

## `cch` 在 Bun 与 Zig 原生层生成

从 Claude Code 二进制中提取的源码快照显示，相关 TypeScript 逻辑位于 `src/constants/system.ts`，入口函数是 `getAttributionHeader(fingerprint: string)`。

函数首先调用 `isAttributionHeaderEnabled()`。如果它返回 `false`，整个 Billing Header 直接返回空字符串。启用 `NATIVE_CLIENT_ATTESTATION` 时，JavaScript 层只在请求中写入固定占位符：

```text
cch=00000;
```

源码注释说明，真正的 `cch` 由 Bun 的 Native HTTP Stack 计算。请求即将发出时，原生层会在已经序列化的请求体中找到 `cch=00000`，再用实际值原位覆盖五个零。占位符与最终值长度一致，因此无需改变 `Content-Length` 或重新分配 Buffer。

Claude Code 使用 Bun 运行 JavaScript。Bun 底层由 Zig 编写，可以把项目编译成不依赖 Node.js 的单文件可执行程序。Anthropic 还维护了资料所示的 `bun-anthropic` 分支；源码注释把原生实现指向：

```text
bun-anthropic/src/http/Attestation.zig
```

这段替换逻辑运行在 JavaScript 引擎的内存空间之外。只在 JavaScript 层拦截 `fetch`、Monkey Patch HTTP 请求，看到的仍是 `cch=00000` 占位符；真正的五位值会在更下层出现。

## 这套 Attestation 解决什么问题

作者将 `cch` 解释为客户端证明机制。Claude Code 使用 OAuth 登录，并把 Token 与 Pro 或 Max 订阅绑定。若其他程序从二进制中提取订阅 Token，再直接调用 API，就可能绕过按 Token 计费的普通接口。

服务器端因此需要判断请求是否确实来自 Claude Code。JavaScript 层只知道固定占位符，Bun 的 Zig 原生层在发送前生成真实 `cch`，服务器再用它验证客户端来源。即使 OAuth Token 正确，缺少有效 Attestation 的请求也可能被拒绝。

这套机制并非不可复现。作者展示的 GitHub 项目 `sub2api` 已经实现相关 `cch` 计算，说明原生层逻辑已经被开源社区分析。该案例只用于说明机制边界，不改变本次问题的处理范围：第三方模型服务不需要 Claude Code 的订阅 Attestation，却可能因为这段变化文本失去 Prompt Cache。

## 关闭 Attribution Header

源码中的 `isAttributionHeaderEnabled()` 会检查 `CLAUDE_CODE_ATTRIBUTION_HEADER`。对于已经明确配置第三方 API、并通过抓包确认 `cch` 导致缓存失效的环境，可以在 `~/.claude/settings.json` 的 `env` 中设置：

```json
{
  "env": {
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0"
  }
}
```

重启 Claude Code 后再次抓包，原来的 `x-anthropic-billing-header` 会从 System Prompt 中消失。作者的复测中，System Block 从三个变为两个，第一个 Block 直接从 Claude Code 身份提示开始，稳定前缀不再因 `cch` 每轮变化。

处理流程可以归纳为：

1. 只在第三方 API 接入场景检查这个问题；
2. 用 `claude-tap` 对比同一 Session 的连续请求，确认 System Prompt 开头是否存在每轮变化的 `cch`；
3. 确认缓存失效与该变化对应后，再设置 `CLAUDE_CODE_ATTRIBUTION_HEADER=0`；
4. 重启 Claude Code 并重新抓包，确认 Billing Header 消失、缓存前缀恢复稳定。

这项设置解决的是第三方服务无法识别 Claude Code Attribution Header 导致的前缀变化，不是所有推理变慢或 Token 增长问题的通用修复。使用 Anthropic 自有服务或订阅 OAuth 路径时，不应从第三方代理案例直接推导出相同处理结论。
