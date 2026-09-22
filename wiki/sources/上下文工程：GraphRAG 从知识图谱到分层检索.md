---
title: 上下文工程：GraphRAG 从知识图谱到分层检索
source: https://www.bilibili.com/video/BV1zoKuzoENM
author: 隔壁的程序员老王
published: 2025-06-26
ingested: 2026-09-04
updated: 2026-09-22
tags:
- AI
- GraphRAG
- 上下文工程
- 应用工程
- 资料摘要
---

# 上下文工程：GraphRAG 从知识图谱到分层检索

原始资料：[[raw/sources/应用工程/上下文与知识工程/AI知识图谱 GraphRAG 是怎么回事？|AI知识图谱 GraphRAG 是怎么回事？]]

## 核心结论

GraphRAG 在原始文本切片与向量检索之外，增加实体、关系、描述、来源映射、社区和分层摘要。Local Search 从实体扩展到原文与邻接关系；本文所述 Global Search 对选定层级的社区报告执行 Map-Reduce，分别生成局部回答再汇总，不逐层下钻原始文本。（[[raw/sources/应用工程/上下文与知识工程/AI知识图谱 GraphRAG 是怎么回事？#图谱查询主线|原文：查询流程]]）

## 构建与查询链路

1. 对原文分块，用大语言模型抽取实体、关系及其描述。
2. 通过 Data Gleaning 反复询问遗漏信息；该过程不能消除模型编造风险。
3. 合并同名实体与分散描述，同时保留图谱到原文的双向映射。
4. 使用 Leiden 社区检测形成层级子图，并由模型生成社区摘要。
5. 将图谱元素、描述、摘要和原文片段分别生成 Embedding，写入向量索引。
6. Local Search 从实体检索并扩展原文、主张、社区报告和邻接关系；Global Search 基于选定层级的社区报告生成局部回答并汇总。

## 限制

- 分块过大容易漏掉局部细节，过小会切断语义关系；图结构用于补充这一矛盾，不等于取消原文分块。
- 实体、关系、描述和额外推断由模型生成，不保证与原文完全一致，仍需来源映射与原文校验。
- 构图、补漏、合并描述和社区摘要均会调用大语言模型，索引成本和 Token 消耗较高。
- 同名实体未必指向同一对象，跨片段合并可能造成实体混淆，需回到原文检查指代。
- Covariate 是带主语、宾语、类型、描述和状态的结构化主张，不是普通片段摘要；作者对其实用价值有限的评价不改变这一数据含义。（[[raw/sources/应用工程/上下文与知识工程/AI知识图谱 GraphRAG 是怎么回事？#4. 协变量（Covariate）抽取|原文：Covariate]]）

## 关联

- 向量 RAG 基础：[[wiki/sources/上下文工程：RAG 从个人知识库到生产检索]]
- 现代 RAG 管道：[[wiki/sources/上下文工程：RAG 从个人知识库到生产检索]]
- GraphRAG 与 Context Rot：[[wiki/sources/上下文工程：第七期 上下文是怎么坏掉的]]
- RAG、GraphRAG 与长期知识资产层：[[wiki/sources/上下文工程：LLM Wiki 的摄取时编译与知识治理]]
- 综合：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
