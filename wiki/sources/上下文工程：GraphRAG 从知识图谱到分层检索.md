---
title: 上下文工程：GraphRAG 从知识图谱到分层检索
source: https://www.bilibili.com/video/BV1zoKuzoENM
author: 隔壁的程序员老王
published: 2025-06-26
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - GraphRAG
  - 上下文工程
  - 应用工程
  - 资料摘要
---

# 上下文工程：GraphRAG 从知识图谱到分层检索

原始资料：[[raw/sources/应用工程/上下文工程/AI知识图谱 GraphRAG 是怎么回事？|AI知识图谱 GraphRAG 是怎么回事？]]

## 核心结论

GraphRAG 在原始文本切片与向量检索之外，增加实体、关系、描述、来源映射、社区和分层摘要。它用 Local Search 从底层实体扩展到原文与邻接关系，用 Global Search 从高层社区摘要向下追溯，分别处理细节定位和全局概括。

## 构建与查询链路

1. 对原文分块，用大语言模型抽取实体、关系及其描述。
2. 通过 Data Gleaning 反复询问遗漏信息；该过程不能消除模型编造风险。
3. 合并同名实体与分散描述，同时保留图谱到原文的双向映射。
4. 使用 Leiden 社区检测形成层级子图，并由模型生成社区摘要。
5. 将图谱元素、描述、摘要和原文片段分别生成 Embedding，写入向量索引。
6. Local Search 从底层实体检索并扩展上下文；Global Search 从高层摘要进入并逐层下钻。

## 限制

- 分块过大容易漏掉局部细节，过小会切断语义关系；图结构用于补充这一矛盾，不等于取消原文分块。
- 实体、关系、描述和额外推断由模型生成，不保证与原文完全一致，仍需来源映射与原文校验。
- 构图、补漏、合并描述和社区摘要均会调用大语言模型，索引成本和 Token 消耗较高。
- 资料把 Covariate 描述为实验性的原文片段文字总结，并保留作者对其实用价值有限的判断。

## 关联

- 向量 RAG 基础：[[wiki/sources/上下文工程：RAG 个人知识库基础架构]]
- 现代 RAG 管道：[[wiki/sources/上下文工程：第四期 RAG 检索增强生成]]
- GraphRAG 与 Context Rot：[[wiki/sources/上下文工程：第七期 上下文是怎么坏掉的]]
- 综合：[[wiki/syntheses/上下文工程：有限窗口中的信息治理]]
