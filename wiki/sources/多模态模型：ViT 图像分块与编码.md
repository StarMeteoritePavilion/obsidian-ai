---
title: 多模态模型：ViT 图像分块与编码
source: https://www.bilibili.com/video/BV1TFG56XERd
author: 隔壁的程序员老王
published: 2026-05-28
ingested: 2026-09-04
updated: 2026-11-01
tags:
  - 视觉与多模态
  - AI
  - 模型原理
  - 多模态模型
  - Vision Transformer
  - ViT
  - 资料摘要
---

# 多模态模型：ViT 图像分块与编码

原始资料：[[raw/sources/模型原理/视觉与多模态/大模型如何看懂图片 ViT Vision Transformer|大模型如何看懂图片 ViT Vision Transformer]]

## 核心结论

Vision Transformer（ViT）先把图片切成固定大小的图块，经线性投影与位置编码形成序列，再由Transformer编码器融合全图关系。原始ViT还加入用于全局分类的Class Token；接入多模态语言模型时，需要按具体架构选择图块特征、池化或其他视觉输出，再通过模态接口与语言表示对齐。

## 从像素到图块特征

一张 $224\times224$ 的彩色图片包含50,176个像素，按RGB通道计为150,528个标量。资料把图片拆成 $16\times16$ 图块，得到 $14\times14=196$ 块，每块包含 $16\times16\times3=768$ 个标量。

局部模型可以从单个图块中提取圆形、透亮、黑点、黄色或毛发等特征，但这些自然语言名称只是教学类比，实际输出是数字向量。Transformer 注意力再把当前图块与其他图块关联，使眼睛、毛发和耳朵等局部信息共同形成更具体的对象特征。

## 编码器与多模态输入

原始ViT在图块序列前加入一个Class Token，用于汇总分类信息。在本例中，196个图块向量加上1个Class Token，组成197个位置的完整序列；每个位置加上位置编码后共同进入Transformer Encoder。若只取图块特征，则对应196行，不能把这个数当作包含Class Token的完整序列长度。与自回归解码器只读当前及此前位置不同，编码器可关联全部有效图块；输出仍是融合上下文后的数字表示，而不是“狗眼睛、狗尾巴”等自然语言标签。（[[raw/sources/模型原理/视觉与多模态/大模型如何看懂图片 ViT Vision Transformer#线性投影与位置编码|投影、位置与Class Token]]、[[raw/sources/模型原理/视觉与多模态/大模型如何看懂图片 ViT Vision Transformer#Transformer Encoder|编码器]]）

视觉输出不能无条件与文本Token拼接。多模态模型未必使用原始ViT的Class Token，可能读取全部图块特征或使用池化；随后还需Projector、Q-Former或Cross-Attention等模态接口完成维度与表征对齐，再与文本共同进入语言模型。具体接口实现与训练细节取决于所用多模态模型。（[[raw/sources/模型原理/视觉与多模态/大模型如何看懂图片 ViT Vision Transformer#多模态接口与语言模型|视觉特征与语言模型的接口]]、[[wiki/sources/多模态模型：架构、数据、推理与检索#架构与数据|接口类型的跨来源对照]]）

## 证据边界与定位

《An Image is Worth 16x16 Words》支持原始ViT的Patch投影、位置Embedding、Class Token与Encoder设计；后续多模态接口类型不属于该论文的统一实现。现有资料没有统一模型、数据划分和评测设置下的准确率比较。视觉编码器也可能采用CNN或混合结构，本例不是所有多模态模型的统一实现。（[[raw/sources/模型原理/视觉与多模态/大模型如何看懂图片 ViT Vision Transformer#来源与版本|来源表与支持范围]]）

## 关联

- 多模态系统全景：[[wiki/sources/多模态模型：架构、数据、推理与检索]]
- Transformer 架构基础：[[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- 注意力机制：[[wiki/sources/模型架构：多头注意力与 QKV]]
- Token 与连续表示：[[wiki/sources/大语言模型：Token、Embedding 与 Latent Space]]
- ViT 用于图像重建：[[wiki/sources/图像重建：DLSS 与 FSR 的时序超分辨率]]
- 模型推理综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
