---
title: 模型架构：PyTorch 手写数字识别
source: https://www.bilibili.com/video/BV1ypUkB7Eki
author: 隔壁的程序员老王
published: 2025-11-27
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型架构
  - PyTorch
  - MNIST
  - 资料摘要
---

# 模型架构：PyTorch 手写数字识别

原始资料：[[raw/sources/模型原理/模型架构/从零搭建神经网络，识别手写数字【PyTorch】【Transformer结构拆解】|从零搭建神经网络，识别手写数字【PyTorch】【Transformer结构拆解】]]

## 核心结论

PyTorch 的 Linear 只要求输入张量最后一个维度与层的输入维度一致，并保持其他维度不变。MNIST 手写数字识别据此把每张 $28\times28$ 图片展平并缩放为 784 个浮点数，再通过 `784→256→128→10` 的三层网络产生 Logits，最后沿类别维执行 Softmax 得到数字 0 至 9 的概率。

## 张量形状与批量计算

`nn.Linear(3, 5)` 的 `weight` 和 `bias` 分别为 $5\times3$ 与 $5$ 形状。形状为 $3$、$2\times3$、$2\times2\times3$ 的输入，会对应产生形状为 $5$、$2\times5$、$2\times2\times5$ 的输出。向量的维度描述一维数组包含多少个数，张量维度描述定位元素需要多少个索引，不能混为一谈。

资料把训练批次写成 $n\times3$，其中 $n$ 是 Batch Size，并指出批量训练可以提升速度。前置维度用于组织单条或多条输入，Linear 只改变最后一个特征维度。

## 数据与模型

MNIST 包含 60000 张训练图片与 10000 张测试图片，每张图片为 $28\times28$ 的八位整数张量，像素值为 0 至 255。`Tensor.view(-1, 784)` 可以在不写死样本数量的情况下展平图片；`float()` 后除以 255，则把输入缩放到 0 至 1。

示例模型使用三个 Linear 和两次 ReLU，将 784 个像素依次映射到 256、128 和 10 个数。新建模型的参数是随机值，本次演示用 `load_state_dict()` 加载预训练参数，只展示推理；训练原理留给合集下一条。

## Logits 与 Softmax 维度

模型输出的十个 Logits 表示图片与十个数字的匹配程度。Softmax 通过指数变换和归一化赋予这些数值概率含义。单张图片的一维输出使用 `dim=0`；三张图片组成 $3\times10$ 输出时使用 `dim=1`，确保每一行在十个类别内独立归一化。这个维度选择取决于类别轴所在位置，不能机械固定。

## 系列定位与关联

官方“从0开始一起学大模型”合集将本文列为第三条。开场明确承接上一条的 Linear 与神经网络原理，结尾预告下一条进入训练；视频没有使用“第三期”标识，因此本库记录合集顺序，不另行命名期数。

- 基础组件：[[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- 下一条训练原理：[[wiki/sources/模型训练：梯度下降与均方误差]]
- PyTorch 训练实战：[[wiki/sources/模型训练：PyTorch 手写数字识别实战]]
- Transformer 总体结构：[[wiki/sources/模型架构：Transformer 编码器、解码器与模型分支]]
- Attention 中的 Softmax：[[wiki/sources/模型架构：多头注意力与 QKV]]
- 后续稀疏 FFN：[[wiki/sources/模型架构：MoE 稀疏专家路由]]
