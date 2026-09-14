---
title: Claude Code权限系统Permission是如何工作的
source: https://www.bilibili.com/video/BV19AEq66Epq
author: 张司机在路上
created: 2026-06-11
tags:
  - AI
  - Claude Code
  - Permission
  - 权限控制
  - Sandbox
  - Prompt Injection
  - 驾驭工程
  - 应用工程
---

# Claude Code权限系统Permission是如何工作的

Claude Code 有时会直接读取文件或执行命令，有时会要求用户确认，即使切换到更宽松的模式也不一定放行。其基本原则是：模型提出动作，运行在用户电脑上的 Claude Code 在执行前检查动作。模型可以请求读取文件、修改代码或运行命令，但不能自行决定工具是否真正执行。

## 权限系统是执行前的门禁

一次工具调用依次经过四个环节：用户提出任务，模型返回 `Read`、`Edit` 或 `Bash(git push)` 等工具调用，Claude Code 权限系统决定允许、询问用户、拒绝或交给 `auto` 分类器，只有通过检查后工具才会执行。

除 `auto` 模式涉及分类器外，资料把基础权限系统描述为写在本地命令行程序中的放行规则，而不是由模型收集风控信号后给动作计算风险分。检查主要读取三类信息：动作所属的工具类型、是否命中用户配置的具体规则，以及本次任务采用的 Permission Mode。

## 工具类型具有不同风险

资料用三类工具解释默认风险差异。

第一类是只读操作，包括 `Read`、`Grep`、`Glob` 和被识别为只读的 Shell 命令。这些动作不直接修改系统，通常不会询问用户。

第二类是 Bash 命令。`npm install`、`git push` 等命令可能安装依赖、联网、删除文件、修改 Git 历史或部署生产环境；没有明确允许时，通常需要询问。

第三类是文件修改，包括 `Edit`、`Write` 等工具。这些动作会修改代码和配置，默认处理也较谨慎。

以上是资料给出的基础分类，不表示只读动作能够读取任意路径，也不表示 Bash 或文件修改必然采用同一种结果；具体规则、模式、工作目录与敏感路径仍会影响最终判断。

## `allow`、`ask` 与 `deny`

权限规则通常写在项目的 `.claude/settings.json` 中，也可以通过 `/permissions` 查看当前项目的规则。规则由工具名与具体匹配条件组成。画面中的配置为：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run test)",
      "Bash(git status)",
      "Read(src/**)",
      "WebFetch(domain:example.com)"
    ],
    "ask": [
      "Bash(git commit *)"
    ],
    "deny": [
      "Bash(git push *)",
      "Read(./.env)"
    ]
  }
}
```

这组示例允许自动运行测试、查看 Git 状态、读取 `src/**` 和抓取 `example.com`；`git commit` 每次询问；`git push` 与读取当前目录的 `.env` 直接拒绝。

资料给出的规则优先级为 `deny > ask > allow`。工具调用一旦命中 `deny`，便直接拒绝，不再由更宽泛的 `allow` 放行。因此，聊天中的提醒或 `CLAUDE.md` 主要指导模型行为；需要长期保持的硬边界应写成权限规则。

## Permission Mode 是本次任务的基线

Permission Mode 定义本次任务的基线策略，`allow`、`ask` 与 `deny` 再在基线上细化。资料列出六种模式。

- `default`：通常放行读取，危险动作询问。
- `acceptEdits`：自动接受工作目录内的文件编辑与常见文件操作。
- `plan`：只读和探索，不直接修改源代码。
- `auto`：把许多待审批动作交给后台 Classifier 判断。
- `dontAsk`：不弹出询问；没有预先允许、原本需要询问的动作直接拒绝。
- `bypassPermissions`：跳过大多数权限提示，只适合隔离环境。

启动时可以使用 `claude --permission-mode acceptEdits` 指定模式，也可以在 `.claude/settings.json` 中设置：

```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

旁白称 `Shift+Tab` 会在 `default`、`acceptEdits`、`plan` 与 `auto` 之间切换；同一画面的表格只在前三项标出“Shift+Tab 循环切换”，`auto` 对应位置显示为横线。资料没有进一步解释这处差异，因此不能确认当时版本如何从界面进入 `auto`。`dontAsk` 与 `bypassPermissions` 则被明确描述为需要显式启动或写入配置。

## `acceptEdits` 只放宽工作目录内的编辑

`acceptEdits` 会自动批准工作目录内的 `Edit`、`Write`、`MultiEdit`，以及资料画面列出的部分文件操作：`mkdir`、`touch`、`rm`、`rmdir`、`mv`、`cp` 和 `sed`。这样可以减少连续修改项目文件时的打断。

放宽范围不等于所有命令都能直接执行。编辑 `src/app.ts` 可能自动批准；编辑 `~/.ssh/config` 会因超出工作目录并涉及敏感配置而提示或阻止；`Bash(npm install)` 仍可能提示，因为它会修改依赖和 Lockfile，还可能联网运行安装脚本；`Bash(git push origin main)` 影响远程仓库，属于明显的外部副作用，也仍可能提示。

## `dontAsk` 用拒绝代替无人值守等待

`dontAsk` 不会把所有动作放开，而是执行已允许和只读的动作，拒绝其他原本需要弹窗确认的动作。资料给出的 CI 配置如下：

```json
{
  "permissions": {
    "defaultMode": "dontAsk",
    "allow": [
      "Read",
      "Grep",
      "Glob",
      "Bash(npm test)",
      "Bash(npm run lint)"
    ],
    "deny": [
      "Bash(git push *)",
      "Bash(rm -rf *)"
    ]
  }
}
```

在此配置中，读取与搜索代码、运行测试和 Lint 可以直接执行；`git push` 与 `rm -rf` 命中 `deny` 后直接失败。`npm install` 没有命中 `allow` 或 `deny`，本来需要询问，但 `dontAsk` 不等待人工确认，因此也会拒绝。这种模式适合 CI、脚本化运行或后台 Agent，而不是更自由的交互模式。

## `auto` 用 Classifier 减少审批疲劳

大量提示最终都由用户机械点击允许，会形成 Approval Fatigue，使提示框失去警示作用。`auto` 的目标是减少这类机械审批，而不是取消权限控制。

资料把粗略流程分为四步：先处理命中的 `allow` 与 `deny`；只读动作和工作目录内的文件编辑通常自动批准；其他动作连同对话上下文交给后台 Classifier；Classifier 拦截后，Claude Code 收到原因并尝试改用更安全的做法。

Classifier 会判断动作是否符合用户请求、是否越权、是否触及危险路径、是否像由 Prompt Injection 驱动，以及风险能否接受。它也会参考对话中的“不要 push”或“部署前先等我 Review”等边界。

聊天约束只是当前上下文中的信息，不是永久权限。对话变长或上下文压缩后，这类约束可能丢失；临时提醒可以留在对话中，硬边界必须写成 `deny`。因此，`auto` 减少的是机械审批，不能替代确定性权限规则。

## `bypassPermissions` 必须配合隔离环境

`bypassPermissions` 的旧启动参数为：

```bash
claude --dangerously-skip-permissions
```

该模式跳过大多数权限提示，让工具调用直接执行。资料强调它只适合容器、Dev Container、虚拟机或可以随时重置的沙盒目录，不应在真实 Home 目录、生产仓库或云环境中随意开启。否则，Prompt Injection、误删文件、误部署与误 Push 更容易造成真实后果。

`bypassPermissions` 表示用户愿意承担沙盒内的风险，不表示动作本身安全。自主实验可以减少人工审批，但必须先隔离环境。

Claude Code 权限系统的核心分工是：模型提出动作，本地客户端按照工具类型、权限规则和 Permission Mode 决定是否放行。下一步若要进一步确认具体实现，还需要检查对应版本的权限系统源码；本资料只讲解基本原理，并未完成源码分析。
