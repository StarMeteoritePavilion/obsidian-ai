---
title: AI 计算硬件：Tensor Core 累加精度与实际提速
source: https://www.bilibili.com/video/BV1bVyizLE5x
author: 隔壁的程序员老王
published: 2026-11-26
ingested: 2026-11-26
updated: 2026-11-26
tags:
  - AI
  - 模型工程
  - 计算硬件
  - GPU
  - NVIDIA
  - 资料摘要
---

# AI 计算硬件：Tensor Core 累加精度与实际提速

原始资料：[[raw/sources/模型工程/计算基础设施/gpu-unlock 让显卡性能翻倍！|gpu-unlock 让显卡性能翻倍！]]

## 核心结论

- **按输入与累加精度比较峰值**。RTX 4090 和 RTX 6000 Ada 同用 AD102，Tensor Core 数量接近。作者以 FP16 输入为例，指出 RTX 4090 的 FP32 累加峰值约为其 FP16 累加峰值的一半。FP32 累加为长和式提供更大的表示范围与精度。见 [[raw/sources/模型工程/计算基础设施/gpu-unlock 让显卡性能翻倍！#为什么矩阵乘法常用 FP16 相乘、FP32 累加|混合精度与两种峰值口径]]。
- **功耗与显存带宽会限制整项任务的提速**。作者给出的 RTX 4090 FP32 累加峰值接近翻倍，ComfyUI 生图和模型微调等任务的提速约为 15% 至 35%，RTX 30 系列的报告区间约为 25% 至 55%。作者用功耗上限、自动降频和显存数据搬运解释峰值与整项任务耗时之间的差距。见 [[raw/sources/模型工程/计算基础设施/gpu-unlock 让显卡性能翻倍！#峰值吞吐如何转化为实际提速|案例与瓶颈]]。
- **最低固件版本会影响安全与兼容性**。来源把固件签名、固件本身的缺陷和硬件是否允许回滚视为三个环节。作者提出以一次性可编程熔丝记录最低固件版本，并指出永久提高版本下限可能影响旧系统或二手显卡的驱动兼容性。见 [[raw/sources/模型工程/计算基础设施/gpu-unlock 让显卡性能翻倍！#固件修补与旧设备兼容性|修补与兼容性讨论]]。

## 数据口径与适用范围

- 视频用 165.6 → 331.6 TFLOPS 描述 RTX 4090 的变化。NVIDIA Ada 架构白皮书列出 RTX 4090 在 FP16 输入下两种默认规格，FP32 累加为 165.2 TFLOPS，FP16 累加为 330.3 TFLOPS，均取表中的非稀疏值。两组数字分别沿用各自来源。[NVIDIA Ada 架构白皮书](https://images.nvidia.com/aem-dam/Solutions/Data-Center/l4/nvidia-ada-gpu-architecture-whitepaper-v2.1.pdf)
- 提速百分比沿用作者举例。来源未列出足以复现实测区间的完整批量、模型版本和驱动条件。本文中的功耗与带宽解释用于定位瓶颈，百分比只适用于来源所述场景。

## 关联

- [[wiki/sources/AI 计算硬件：内存带宽、互联与软件生态]]
- [[wiki/syntheses/模型推理：从 Token、Latent 到多模态交错思维#推理慢、显存高、费用高时先查什么|推理的性能与成本排查]]
