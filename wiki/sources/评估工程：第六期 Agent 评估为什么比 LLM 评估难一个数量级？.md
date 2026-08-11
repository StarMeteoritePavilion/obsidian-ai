---
title: 评估工程：第六期 Agent 评估为什么比 LLM 评估难一个数量级？
source: https://www.bilibili.com/video/BV1URui6CEBT
author: 晴天AI实战
published: 2026-08-11
ingested: 2026-08-11
tags:
  - AI
  - 评估工程
  - 应用工程
  - 资料摘要
---

# 评估工程：第六期 Agent 评估为什么比 LLM 评估难一个数量级？

原始资料：[[raw/sources/应用工程/评估工程/第六期：Agent 评估为什么比 LLM 评估难一个数量级？|第六期：Agent 评估为什么比 LLM 评估难一个数量级？]]

## 核心结论

LLM 评估主要检查单次输出文本，Agent 评估必须检查多轮工具调用最终改变了什么状态。表达正确不等于任务完成，真正的评估对象是文件、数据库、页面和后端等环境中的 Outcome。

错误传播、非确定性和环境依赖共同扩大了评估复杂度。资料因此把 Agent 评估定义为独立问题，而不是把 LLM 裁判多运行几次。

## 五层架构与八个术语

五层架构由 Suite、Harness、Graders、Metrics 和 Agent 类型适配组成。Suite 组织任务，Harness 隔离环境并执行、记录和聚合，Graders 判断结果，Metrics 处理多次运行，类型适配层为不同 Agent 组合评分器。

共同词汇表包括 Task、Trial、Grader、Transcript、Outcome、Harness、Suite 和 Agent Scaffold。关键区分是：Transcript 保存过程，Outcome 表示环境最终状态；前者用于诊断，后者用于判定任务是否真正完成。

## Capability、Regression 与双指标

Capability Eval 从低通过率出发，量化 Agent 获得新能力的过程；能力达到稳定高通过率后进入 Regression Suite，用于检查 Prompt、模型或依赖变更是否破坏既有能力。资料把两者概括为“Capability 爬山，Regression 护栏”。

`pass@k` 衡量 k 次尝试中至少一次成功的概率，适合“一次成功就够”的产品；`pass^k` 衡量 k 次尝试全部成功的概率，适合要求每次可靠的产品。两项指标在 k 等于 1 时相同，k 增大后方向相反，因此指标选择也是产品责任的选择。

## 评分器铁律

评分器应检查产出，不检查路径。规定 Agent 必须调用某组工具，会把实现方式误当成业务目标；更可靠的标准是单元测试是否通过、退款记录是否创建、配置是否正确改变。

Code-based 评分器适合确定性的编程结果，Model-based 评分器适合对话和研究中的语义判断，Human Grader 提供金标准和校准。三者可以组合，但自动评分器必须继续接受人工抽检。

## 四类 Agent 的不同配方

- **Coding Agent**：以确定性单元测试为核心，同时检查 fail-to-pass 与 pass-to-pass。
- **Conversational Agent**：由第二个 LLM 模拟用户发起多轮对话，同时进行 State check 和 LLM rubric 评分。
- **Research Agent**：组合 Groundedness、Coverage 与 Source Quality，处理没有唯一标准答案的研究任务。
- **Computer Use Agent**：在 GUI 沙箱中检查 URL、页面状态和后端数据。

四类 Agent 共用评估框架，但不能共用一套固定评分器。任务的最终状态、风险和可验证性决定评分组合。

## 八步落地路线

资料给出的路线依次是：用 20～50 个典型任务起步；手动运行全部任务；写清通过与失败标准；覆盖高频、边缘和已知失败；让每个 Trial 从干净环境启动；确定性评分优先、模型补位、人工校准；阅读 Transcript 而不只看总分；让成熟 Capability 进入 Regression 并持续增加难题。

环境隔离是其中的关键约束。共享文件、数据库或 Git 历史会让不同 Trial 相互泄漏信息，评估结果因此无法独立复现。

## 适用边界

视频中的“高一个数量级”是对错误传播、非确定性和环境依赖叠加后的复杂度概括，不是跨项目通用的成本倍数。SWE-bench Verified 从约 40% 到 80% 以上、第一周 20 个任务到第六个月 500 多个任务，也都是本期资料展示的案例和路线示例。

`pass@k` 与 `pass^k` 不能互相替代。前者反映搜索宽容度，后者反映连续可靠性；只报告其中一项，会掩盖另一种产品风险。

## 系列定位

开场画面明确标注“评估工程系列 · 第 6 期 / 共 8 期”。本期资料承接第五期对产品级评估系统的讨论，把评估对象从单次模型回复扩展到 Agent 的多轮行为和环境状态；结尾预告第七期“生产护栏”。

## 关联

- 上一期：[[wiki/sources/评估工程：第五期 能跑的评估和能放心用的评估，差在哪一层？]]
- 跨资料综合：[[wiki/syntheses/评估工程：从通用基准到业务质量门]]
- Agent 循环与独立评判器：[[wiki/syntheses/循环工程：从逐轮操作到外部调度]]
