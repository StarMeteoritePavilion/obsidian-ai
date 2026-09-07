---
title: 模型训练：PyTorch MNIST 完整实践
source:
  - https://www.bilibili.com/video/BV1ypUkB7Eki
  - https://www.bilibili.com/video/BV1SdBcB7EtG
author:
  - 隔壁的程序员老王
published: 2025-11-27
ingested: 2026-09-04
updated: 2026-09-07
tags:
  - AI
  - 模型工程
  - PyTorch
  - MNIST
  - 资料摘要
---

# 模型训练：PyTorch MNIST 完整实践

原始资料：[[raw/sources/模型工程/训练与后训练/用 PyTorch 搭建并训练 MNIST 手写数字识别模型|用 PyTorch 搭建并训练 MNIST 手写数字识别模型]]

## 合并关系

当前原文合并了“模型结构、张量与推理”和“完整训练循环”两个来源。本摘要整合原两份摘要的独有结论，并以当前合并稿的运行证据为准。

## 核心结论

- MNIST图片展平为784维，网络为`784→256→128→10`；输出是Logits，`CrossEntropyLoss`直接接收Logits。
- 训练使用Dataset、DataLoader、梯度清零、反向传播和Adam更新；测试必须使用独立测试集，并同时设置`eval()`和`no_grad()`。
- 本次CPU、float32、种子42实测跑满5个Epoch，每轮60000张训练图；在10000张测试图上得到平均损失0.14691142849624156、正确9635张、准确率96.35%。
- 保存`state_dict`后加载到同架构模型，固定1000张测试图的类别完全一致，Logits最大绝对误差为0.0。

## 限制

结果绑定Python 3.14.4、PyTorch 2.14.0、torchvision 0.29.0与CPU环境；未执行MPS训练。原资料缺少可复核权重的单图高置信度结果未沿用。

关联：[[wiki/sources/模型训练：梯度下降与均方误差]]、[[wiki/syntheses/深层模型训练稳定性：残差、更新与路由]]。
