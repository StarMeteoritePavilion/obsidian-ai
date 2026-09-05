---
title: 上下文工程：LLM Wiki 的摄取时编译与知识治理
source: https://www.bilibili.com/video/BV1NG9xBUEju
author: 唐国梁Tommy
published: 2026-04-29
ingested: 2026-09-04
updated: 2026-09-05
tags:
  - 上下文与知识工程
  - AI
  - 应用工程
  - 上下文工程
  - LLM-Wiki
  - RAG
  - GraphRAG
  - 知识工程
  - 资料摘要
---

# 上下文工程：LLM Wiki 的摄取时编译与知识治理

原始资料：[[raw/sources/应用工程/上下文与知识工程/RAG vs GraphRAG vs LLM Wiki 一次讲透：Karpathy引爆的LLM Wiki，是知识工程的下一次革命，还是又一个被高估的"自我进化"|RAG vs GraphRAG vs LLM Wiki 一次讲透]]

## 核心结论

LLM Wiki 把跨资料综合从查询阶段前移到摄取阶段：原始资料进入后立即编译为结构化、可导航、可持续更新的知识页面，后续查询复用已经建立的结论与链接。它不替代 RAG 或 GraphRAG，而是增加长期知识资产层；RAG 继续承担局部召回，GraphRAG 补充实体关系和全局结构。

## 架构与运行控制面

系统分为不可变的 Raw sources、可更新的 Wiki 和规定命名、引用与冲突处理方式的 Schema。`index.md`、`log.md`、`overview.md`、`hot.md`、`purpose.md`、`state/`、`review/queue` 与 `graph.json` 组成运行控制面，使模型能够导航内容、记录变更、缓存增量状态并把高风险判断交给人工审核。

知识页可以包括 Source、Entity、Concept、Comparison、Question、Synthesis、Decision、Gap 和 Meta。Frontmatter 是结构化治理的基础；资料展示的 `confidence`、`provenanceState`、`contradictedBy` 和 `inferredParagraphs` 进一步区分来源覆盖、冲突与模型推断。

## 增量编译与持续运行

一次 Ingest 可能更新 8～15 个页面。八步管线依次接收和规范化原始材料、读取 Schema、分析与生成页面、建立交叉引用、更新控制面并进入 Review Queue。`llm_wiki` 用两次 LLM 调用分离结构分析与页面生成；`atomicmemory` 通过 SHA-256 Hash 只重新编译变化部分。

Ingest 之后还需要四项动作：Query 查询编译结果；Save／Crystallization 把高价值对话写回知识库；Lint 分别检查结构问题和语义问题；Research 从知识缺口发起外部研究。缺少 Save，系统会退化为带日志的聊天工具；缺少 Lint 与 Review，错误会沿交叉链接持续传播。

## 检索、对抗与治理

大规模 Wiki 可以组合 BM25、Vector 与 Graph，再用 RRF 融合结果。资料主张按查询类型而不是语料体量路由：Single-hop 事实检索优先 Vanilla RAG，多跳、跨实体和全局理解再使用 GraphRAG，长期综合结果由 LLM Wiki 保存。

`nvk/llm-wiki` 的 Thesis Mode 为争议命题派出 5～10 个立场分化的 Agent 并行取证，输出 `supported`、`partially supported`、`contradicted`、`insufficient evidence` 或混合证据。该方案增加数倍 Token 与延迟，只适合高风险或需要主动破除共识偏差的结论。

长期治理依赖 Confidence 与 Supersession：未验证内容逐步降权，新结论取代旧结论时保留来源、时间和适用范围。最大风险是幻觉回写闭环，即派生页面中的错误在后续 Ingest 中被当作事实继续扩展。段落级 Provenance、定期 Lint 和 Confidence 只能缓解，高风险声明仍需进入 Review Queue。

## 边界与定位

Agent Memory 保存会话、任务过程和执行轨迹，LLM Wiki 沉淀可长期维护的结论；Obsidian 与 Notion 更接近前端容器，Schema、Ingest、Lint、Review 和 Save-back 才构成知识维护工作流。Graph 适合导航和影响分析，Page 适合解释、引用和人工审校，二者应共存。

Wiki 还可以通过 `llms.txt`、JSON-LD、GraphML 和 MCP Resources 分别向其他 Agent 提供自然语言说明、结构化语义、图谱交换和资源接口。启用外部 Research、Web Search 或云模型后，内部知识可能离开本地边界，因此本地优先、访问控制、审计日志和数据驻留属于架构约束，而不是上线后的附加项。

长期生命周期需要 Retention 与 Forgetting：旧资料不必直接删除，但长期未验证、很少访问或被新证据削弱的内容，应在检索排序和上下文注入中逐步降权。资料提出 Source Coverage、Index Freshness、Citation Coverage、Review Backlog 和 Query Save Rate 五项健康指标，分别检查来源覆盖、索引更新、引用覆盖、待审积压和高价值回答回写。

官方标题、开场和结尾均无期数标识，单 P 标题与视频标题一致；官方“前沿论文与最新技术趋势洞察”合集中的第 13 个位置只表示收录顺序。因此定位为应用工程／上下文与知识工程独立专题，不编排期数。

## 关联

- 基础向量检索：[[wiki/sources/上下文工程：RAG 个人知识库基础架构]]
- 生产 RAG：[[wiki/sources/上下文工程：第四期 RAG 检索增强生成]]
- 图结构与分层检索：[[wiki/sources/上下文工程：GraphRAG 从知识图谱到分层检索]]
- 上下文工程综合：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
- Agent 记忆边界：[[wiki/sources/Agent 记忆：MemRL 运行时强化学习]]
