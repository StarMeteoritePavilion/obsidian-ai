---
title: Claude Code：Skill 渐进式披露与第三方执行边界
source: https://www.bilibili.com/video/BV19bjN61EaK
author: 张司机在路上
published: 2026-06-15
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Anthropic
  - Skill
  - 渐进式披露
  - Prompt Caching
  - ToolSearch
  - 上下文工程
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：Skill 渐进式披露与第三方执行边界

原始资料：[[raw/sources/应用工程/驾驭工程/你装的skill都是如何被Claude Code识别和加载的|你装的skill都是如何被Claude Code识别和加载的]]

## 核心结论

资料中的 Claude Code Skill 采用三层渐进式披露：会话开始时只注入名称和 frontmatter 描述；模型通过 `Skill` 工具点名后，客户端才读取并注入处理过的 `SKILL.md`；主说明引用的参考文档和脚本则按任务需要继续加载。Skill 数量增加不会让全部说明书同时进入初始上下文。

## 加载链路

PDF 案例先在 System Prompt 中暴露 `document-skills:pdf` 的名称与适用描述。模型判断用户需要分析 *Attention Is All You Need* 后，返回一次调用 `Skill` 的 `tool_use`。

下一次请求中的 `tool_result` 只包含 `Launching skill: document-skills:pdf`，处理后的 `SKILL.md` 以同一用户消息中的独立 Text Block 注入。处理包括去掉 frontmatter、添加 `Base directory for this skill:`，并在末尾拼接 `ARGUMENTS:`。目录用于解析 `REFERENCE.md`、`FORMS.md` 和脚本的相对路径；当前任务没有读取接近三万字符的两份参考文件，而是依主说明运行 `pdftotext`。

## 与 ToolSearch 的边界

Skill 清单提供名称和描述，Deferred Tool 清单只提供名称。Skill 正文追加在 `messages` 末尾，不改变既有缓存前缀；ToolSearch 的完整 Schema 进入 `tools` 数组，需要 `defer_loading` 与 `tool_reference` 共同保持可见性和 Prompt Cache 前缀。

两者都多一次“索取说明书”的模型往返，但信任边界不同：ToolSearch 暴露官方工具实现，Skill 会执行第三方提示词和脚本。资料简介中的恶意教学样例分别尝试外传凭据和原地加密项目文件，说明渐进式加载只减少上下文开销，不负责判断来源是否可信。

## 证据边界

以上链路来自作者当次 Claude Code 抓包和 PDF Skill 案例，不应外推为所有版本、其他 Agent 或全部 Skill 运行时的固定协议。资料能够确认客户端当次如何组装消息，但没有提供对应 Claude Code 版本号、完整抓包文件或可复核代码实现。

## 关联

- 上一篇：[[wiki/sources/Claude Code：权限规则、Permission Mode 与本地放行]]
- 工具按需加载：[[wiki/sources/Claude Code：ToolSearch 延迟加载与缓存保持]]
- 请求中的 Skill 清单：[[wiki/sources/Claude Code：请求结构、SSE 与缓存 Token 计量]]
- Skill 变更与缓存：[[wiki/sources/Claude Code：模型、工具、注入与 TTL 的缓存命中边界]]
- 上下文治理：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
