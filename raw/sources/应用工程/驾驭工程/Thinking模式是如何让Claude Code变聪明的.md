---
title: Thinking模式是如何让Claude Code变聪明的
source: https://www.bilibili.com/video/BV1fc5j62E1Q
author: 张司机在路上
created: 2026-05-10
tags:
  - AI
  - Claude Code
  - Anthropic
  - Thinking
  - Reasoning
  - Transformer
  - 驾驭工程
  - 应用工程
---

# Thinking模式是如何让Claude Code变聪明的

Claude Code 的 `/config` 中提供 Thinking Mode 开关，Claude 桌面应用也会在正式回答前显示一段思考过程。这个模式改变的不是 Transformer 预测下一个 Token 的基本机制，而是 API 输出结构与可用 Token 预算：模型可以先生成中间推理，再据此生成行动或正式答案。

## 从上下文直接行动，到先推理再行动

普通模式可以概括为“上下文 → 行动”。Claude Code 每轮向模型发送系统提示词、`CLAUDE.md`、工具说明和用户问题等上下文，模型读取后直接生成回复、工具调用、命令或文件修改。

这种方式足以处理把按钮文案从 A 改成 B 等简单任务。任务变复杂后，直接行动更容易只关注最显眼的代码。例如，用户偶尔遇到请求丢数据，原因可能位于缓存、网络或并发冲突；模型需要先列出可能性，再逐项排除。

Thinking 模式可以概括为“上下文 → 中间推理 → 行动”。API 允许模型在最终输出前生成一段 Thinking Token，用于梳理假设、排除错误方向并定位问题，再生成行动方案。这不是在 Claude Code 外额外套用一个算法，而是模型的 API 输出结构和 Token 预算发生了变化。

Transformer 生成下一个 Token 时，会把此前 Token 作为上下文。先生成的 Thinking Token 因而会进入后续生成过程，而不是只供界面展示的装饰。它相当于模型在行动前写给自己的草稿。

## TTL 缓存案例

案例包含两个 JavaScript 文件。`cache.js` 实现带 TTL 的内存缓存，提供 `set` 和 `get`；`app.js` 写入键 `user-1`，把 TTL 设为一秒，并在两秒后读取。预期结果为 `null`，实际却仍能读到值。

模型在 Thinking Block 中把问题定位到 `get`：函数取出了缓存值，却没有检查过期时间，因此这个缓存实际上只像一个普通的 `Map`，过期行为从未发生。

随后，Text Block 给出修复方案：在 `get` 中比较 `Date.now()` 与 `entry.expiry`，条目过期后从 `Map` 删除并返回 `null`。这种读取时清理过期条目的方式称为 Lazy Eviction，即惰性清理。

这个案例中，Thinking Block 完成问题定位，Text Block 根据定位结果给出修复。正式答案不是模型看到输入后的第一反应，而是已经经过一轮显式推理的结果。

## `thinking` 与 `effort` 控制什么

抓包请求中有两个字段控制 Thinking 行为。

`thinking` 的值为 `adaptive`。它不是要求每次都开启思考，而是让模型根据当前问题的复杂度自行决定是否思考以及思考深度。

`effort` 的值为 `high`，用于指定大致的推理强度。资料列出五档：`low`、`medium`、`high`、`xhigh` 和 `max`。档位越高，允许的思考 Token 越多，回答可以更细，但延迟也更高；档位越低，思考步骤更少，返回更快。

## 响应中的 Thinking Block 与 Text Block

资料把普通模式的响应概括为只包含 `type: text` 的答案 Block。Thinking 模式下，`content` 数组在 Text Block 前增加一个 `type: thinking` 的 Block，用于保存模型生成的推理草稿。

两种模式仍使用同一套自回归生成机制。区别在于，Thinking 模式会先生成 Thinking Block，后续 Text Block 再基于原始输入和已经生成的推理内容继续输出。因此，Thinking Block 为正式回答提供中间计算步骤。

## 推理习惯怎样形成

资料把大模型训练概括为四个阶段：预训练、指令微调、基于人类反馈的强化学习（RLHF）和 Reasoning Tuning。前三个阶段分别让模型学习语言、跟随指令并更符合人类偏好；推理模型还通过 Reasoning Tuning 学习先展开推理、再给出答案的输出习惯。

以方程 $2(x-3)=14$ 为例，普通指令微调样本可以直接给出 $x=10$。推理训练样本则先在 Reasoning 标签中写出步骤：展开括号，两边加 6 得到 $2x=20$，再同时除以 2 得到 $x=10$，最后才在 Answer 标签中给出答案。

模型反复学习这类逐步拆解样本后，会形成先写推理再回答的输出方式。运行时还需要 API 协议提供相应输出位置；案例中的 `thinking: adaptive` 允许模型在 `content` 中生成 Thinking Block。

Thinking Mode 的收益是让模型在复杂任务中枚举可能性、排除错误方向，再形成答案；代价是增加输出延迟和 Token 消耗。
