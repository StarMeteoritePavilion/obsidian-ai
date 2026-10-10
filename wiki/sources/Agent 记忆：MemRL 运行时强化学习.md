---
title: Agent 记忆：MemRL 运行时强化学习
source: https://www.bilibili.com/video/BV18twWzuELh
author: 唐国梁Tommy
published: 2026-03-14
ingested: 2026-09-03
updated: 2026-10-10
tags:
- AI
- 应用工程
- 记忆工程
- Agent
- 强化学习
- 资料摘要
---

# Agent 记忆：MemRL 运行时强化学习

原始资料：[[raw/sources/应用工程/AI Agent/AI Agent终于能边用边学了？MemRL这篇论文太有工程价值了！MemRL给出新解法，提出无参数化Runtime Learning|冻结权重的记忆效用学习：MemRL 无参数化 Runtime Learning]]

## 核心结论

MemRL 在冻结语言模型权重的情况下，把环境反馈写入情景记忆的 Q-value，使 Agent 能在运行时调整未来的记忆选择。它针对的是静态语义检索无法区分“语义相关”与“功能有用”，以及持续 Fine-tuning 成本高、可能发生灾难性遗忘的问题。

## 方法

系统先按余弦相似度粗筛 Top-K 记忆，再用历史环境反馈学习到的 Q-value 精排。每条记忆保存意图、经验和效用分数；选中记忆注入 Prompt 后，冻结模型生成 Action，环境奖励再通过蒙特卡洛规则更新 Q-value，并把当前轨迹摘要为新记忆。

这种结构把稳定推理与可塑记忆分离：模型不做梯度更新，检索—更新控制器负责选择、反馈和写回。它与普通 RAG 的区别不在于是否使用向量检索，而在于历史执行结果会持续改变候选记忆的排序。

## 实验与限制

实验覆盖 HLE、BigCodeBench、ALFWorld 和 Lifelong Agent Bench；相对 MemP 基线的累计成功率提高 3.8 个百分点，在 ALFWorld 与 OS 任务等场景最高提高 6.2 个百分点。数字限于来源记载的实验设置，未独立复现。（[[raw/sources/应用工程/AI Agent/AI Agent终于能边用边学了？MemRL这篇论文太有工程价值了！MemRL给出新解法，提出无参数化Runtime Learning#实验结果与边界|实验与指标归属]]）

冻结语言模型权重不等于零额外推理成本；现有资料没有成本对照表，不能量化检索、记忆更新与轨迹摘要的额外开销。

系统仍依赖可靠奖励，记忆库持续增长还需要规模管理、淘汰和冲突治理。当前讲解面向单 Agent；多 Agent 共享记忆时，错误经验可能跨实例污染决策。

## 关联

- 应用层上下文与外部记忆：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- 参数强化学习与行为选择：[[wiki/syntheses/大模型后训练：从模仿到行为选择]]
- 外部技能库路线：[[wiki/sources/大模型后训练：SKILLRL 技能增强强化学习]]
- 模型内部长序列记忆：[[wiki/sources/长序列建模：Memory Caching]]
