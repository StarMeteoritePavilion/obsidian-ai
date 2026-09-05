---
title: 模型训练：PyTorch 手写数字识别实战
source: https://www.bilibili.com/video/BV1SdBcB7EtG
author: 隔壁的程序员老王
published: 2025-12-25
ingested: 2026-09-04
updated: 2026-09-04
tags:
  - AI
  - 模型训练
  - PyTorch
  - MNIST
  - CrossEntropyLoss
  - 资料摘要
---

# 模型训练：PyTorch 手写数字识别实战

原始资料：[[raw/sources/模型工程/训练与后训练/30行代码训练模型 手写数字识别【PyTorch实战】|30行代码训练模型 手写数字识别【PyTorch实战】]]

## 核心结论

PyTorch 训练链路把 Dataset 的单样本预处理、DataLoader 的批量组织、模型前向计算、CrossEntropyLoss、梯度清零与反向传播、参数更新和权重保存连接起来。模型输出层直接产生 Logits；`CrossEntropyLoss` 已包含 Softmax 对应的处理，不能在输出层重复添加 Softmax。

## 数据预处理与批量加载

MNIST 原始图片为 $28\times28$ 的整数张量。`transform` 函数接收 `PIL.Image.Image`，用 `functional.to_tensor()` 转回张量并缩放到 0 至 1，再用 `view(28 * 28)` 展平。与直接读取全部 `data` 属性相比，Dataset 把预处理放在每次返回样本时执行，更适合无法整体载入内存的数据。

示例 DataLoader 使用 `batch_size=50` 与 `shuffle=True`，每轮返回形状为 $50\times784$ 的图片张量和形状为 $50$ 的标签张量。Batch Size 属于人为设置的超参数。

## 损失与训练循环

资料用 $Loss=-\ln(p)$ 解释 CrossEntropyLoss，其中 $p$ 是正确类别的概率。训练外层执行五个 Epoch，内层遍历 DataLoader；每个批次先计算 Logits 和 Loss，再清空已有梯度、调用 `loss.backward()`，最后更新参数。

手动更新时需要放入 `torch.no_grad()`，并跳过 `param.grad is None` 的参数。后者可能来自有意冻结，或参数在当前计算中没有对损失产生贡献。实际工程通常用 Optimizer 代替逐参数更新；示例使用 `torch.optim.Adam`、初始 Learning Rate 0.01，以及 `optimizer.zero_grad()`、`loss.backward()`、`optimizer.step()` 三步。

## 验证、保存与边界

示例以训练集第 90 条、标签为 6 的图片演示识别结果，但明确指出实际工程应使用测试集验证，避免只根据训练数据判断模型效果。权重通过 `torch.save(mnist_model.state_dict(), "output.pth")` 保存，再以 `load_state_dict()` 和 `torch.load(..., weights_only=True)` 加载。

Batch Size 50、Epoch 5、Learning Rate 0.01、模型 `784→256→128→10` 及训练集样本预测结果只属于本次教学实验，不构成其他任务的默认配置、精度或泛化保证。资料将动态调整步幅与 Optimizer 放在同一段讲解；Adam 示例证明 Optimizer 可以替代手动更新，不表示所有 Learning Rate 调度都由 Optimizer 本身完成。

## 系列定位与关联

官方“从0开始一起学大模型”合集将本文列为第五条。开场明确承接此前的 PyTorch 模型结构和梯度下降原理，正文完成手写数字识别训练实现；视频没有使用“第五期”标识，因此本库记录合集顺序，不另行命名期数。

- 模型与数据预处理：[[wiki/sources/模型架构：PyTorch 手写数字识别]]
- 梯度下降原理：[[wiki/sources/模型训练：梯度下降与均方误差]]
- Linear 与 MLP：[[wiki/sources/模型架构：Linear、Activation 与 MLP]]
- 参数冻结与 LoRA：[[wiki/sources/大模型微调：LoRA 低秩适配]]
- 模型结构与训练综合：[[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维]]
