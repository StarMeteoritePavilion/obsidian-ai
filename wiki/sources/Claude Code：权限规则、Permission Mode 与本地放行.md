---
title: Claude Code：权限规则、Permission Mode 与本地放行
source: https://www.bilibili.com/video/BV19AEq66Epq
author: 张司机在路上
published: 2026-06-11
ingested: 2026-09-14
updated: 2026-09-14
tags:
  - AI
  - Claude Code
  - Permission
  - 权限控制
  - Sandbox
  - Prompt Injection
  - 驾驭工程
  - 应用工程
  - 资料摘要
---

# Claude Code：权限规则、Permission Mode 与本地放行

原始资料：[[raw/sources/应用工程/驾驭工程/Claude Code权限系统Permission是如何工作的|Claude Code权限系统Permission是如何工作的]]

## 核心结论

Claude 模型负责提出 `Read`、`Edit` 或 `Bash` 等动作，运行在用户电脑上的 Claude Code 在真正执行前检查工具类型、权限规则与 Permission Mode。`CLAUDE.md` 和聊天约束主要指导模型；`allow`、`ask`、`deny`、模式与更底层的 Sandbox 承担执行边界。

## 权限规则

- 项目规则通常写在 `.claude/settings.json` 的 `permissions` 中，也可以通过 `/permissions` 查看。
- `allow` 自动允许，`ask` 要求询问，`deny` 直接拒绝；资料给出的优先级为 `deny > ask > allow`。
- 画面中的规则示例精确包含 `Bash(npm run test)`、`Bash(git status)`、`Read(src/**)`、`WebFetch(domain:example.com)`、`Bash(git commit *)`、`Bash(git push *)` 与 `Read(./.env)`。
- 只读工具通常放行，但仍受路径和具体规则限制；Bash 与文件修改可以产生依赖、网络、文件、Git 历史、部署等副作用，默认更谨慎。

## 六种 Permission Mode

`default` 通常放行读取、询问危险动作；`acceptEdits` 自动批准工作目录内的编辑与部分文件操作；`plan` 只读探索；`auto` 把许多待审批动作交给后台 Classifier；`dontAsk` 不弹窗，未预先允许且原本需要询问的动作直接拒绝；`bypassPermissions` 跳过大多数提示，只适合隔离环境。

模式可由 `claude --permission-mode acceptEdits` 或 `permissions.defaultMode` 指定。旁白称 `Shift+Tab` 可以在 `default`、`acceptEdits`、`plan` 与 `auto` 间切换，但同一画面只为前三项标注“Shift+Tab 循环切换”，`auto` 显示横线；资料没有解释这处冲突。

## 三种关键边界

`acceptEdits` 放宽的是工作目录内编辑。资料画面列出 `Edit`、`Write`、`MultiEdit` 以及 `mkdir`、`touch`、`rm`、`rmdir`、`mv`、`cp`、`sed`；`~/.ssh/config`、`npm install` 和 `git push` 仍可能提示或阻止。

`dontAsk` 面向 CI 与后台 Agent。已允许操作可以执行，命中 `deny` 的操作直接失败，没有规则且原本需要询问的 `npm install` 也直接拒绝。它解决无人值守流程不能等待弹窗的问题，不代表权限更宽。

`auto` 先处理明确的 `allow` 与 `deny`，通常自动批准只读动作和工作目录内编辑，再把其他动作及上下文交给 Classifier。对话里的“不要 push”可能参与判断，却会随对话变长或上下文压缩而丢失；硬边界仍应写成 `deny`。

`bypassPermissions` 的旧参数为 `claude --dangerously-skip-permissions`。它适用于容器、Dev Container、虚拟机和可重置沙盒，不适用于真实 Home 目录、生产仓库或云环境。跳过提示只表示用户接受沙盒内风险，不证明动作安全。

## 证据边界

资料明确声明本篇只解释基本原理，不分析权限源码，也没有提供 Claude Code 版本号、完整规则实现或 Classifier 评测。工具分类、模式行为和检查顺序来自作者讲解与画面，不能外推为所有版本的固定实现；`auto` 的界面切换方式还存在上述旁白与表格冲突。

## 关联

- 上一篇：[[wiki/sources/Claude Code：compact 上下文压缩与工作现场恢复|Claude Code：/compact 上下文压缩与工作现场恢复]]
- 下一篇：[[wiki/sources/Claude Code：Skill 渐进式披露与第三方执行边界]]
- 工具调用闭环：[[wiki/sources/Claude Code：tool_use、tool_result 与客户端工具闭环]]
- Agent 行动与安全：[[wiki/syntheses/AI Agent：从工具调用到可信行动]]
- Agent Harness：[[wiki/syntheses/驾驭工程：模型之外的 Agent Harness]]
