---
title: "用 PyTorch 搭建并训练 MNIST 手写数字识别模型"
source:
  - "https://www.bilibili.com/video/BV1ypUkB7Eki"
  - "https://www.bilibili.com/video/BV1SdBcB7EtG"
author:
  - "隔壁的程序员老王"
created: 2025-11-27
tags:
  - "AI"
  - "模型架构"
  - "PyTorch"
  - "MNIST"
  - "神经网络"
  - "模型训练"
  - "CrossEntropyLoss"
updated: 2026-09-07
---

# 用 PyTorch 搭建并训练 MNIST 手写数字识别模型

一个前馈分类模型的完整链路是：把图像变成张量，计算十个类别的分数，用损失函数训练，再在独立测试集上评估，最后保存参数并检查重新加载后的输出。本文用同一套 `784→256→128→10` 网络走完这条链路，配套代码实际训练五轮，测试集准确率为 **96.35%**。这是本次 CPU 环境的结果，不代表所有设备或训练配置。[^实测]

## MNIST 与张量形状

本次加载的 MNIST 训练集有60,000张图，测试集有10,000张图，每张为28×28的灰度图。`torchvision.datasets.MNIST` 的 `train=True`、`train=False` 分别选择两部分；脚本对样本数、原图形状、`torch.uint8` 类型及0至255的像素范围作了实际断言。[^数据]

向量的维数与张量的轴数要分开：包含五个数的向量形状为 `[5]`，在 PyTorch 中是一个一维张量；`[1, 5]` 是二维张量。`nn.Linear(3, 5)` 的权重形状是 `[5, 3]`，偏置为 `[5]`。它接受任意前导维度，但最后一维必须等于3，并仅将这一维替换为5，例如 `[2, 2, 3]→[2, 2, 5]`。[^接口]

## 展平归一化与批量加载

`to_tensor` 把本例的 PIL 灰度图转成 `[1, 28, 28]` 的浮点张量，并把八位像素除以255，缩放到 `[0, 1]`；随后 `flatten()` 得到784个特征。该缩放只改变数值尺度，不代表所有输入格式都会被 `to_tensor` 自动归一化。[^数据]

也可以用 `view(-1, 784)` 指定形状，其中最多一个 `-1` 可由元素数推算；但元素数相同并不足够，原张量的步长必须满足该视图的兼容条件。非连续张量不一定能直接 `view`。本文用 `flatten` 避免把任何张量都可任意改形状当作前提。[^接口]

Dataset 按索引提供样本，`transform` 在取出图片时进行预处理；MNIST 自身仍会加载其原始数据，不能由 Dataset 抽象推断所有数据集都会按需从磁盘流式读取。DataLoader 把训练样本随机组成批次；本次训练批量50，测试批量1000，测试不打乱。[^数据]

| 阶段 | 批量形状 | 含义 |
| --- | --- | --- |
| 预处理后输入 | `[batch, 784]` | 每行一张图片的浮点像素 |
| 第一层及ReLU | `[batch, 256]` | 第一层特征 |
| 第二层及ReLU | `[batch, 128]` | 第二层特征 |
| 最后一层 | `[batch, 10]` | 数字0至9的Logits |
| 标签 | `[batch]` | 各图的整数类别 |

## 模型与 Logits

模型依次为三个 Linear、两个 ReLU，最后不加 Softmax。Logits 是未经归一化的类别分数。用于显示时可计算：

$$
\operatorname{softmax}(z_i)=\frac{e^{z_i}}{\sum_j e^{z_j}}
$$

对形状 `[10]` 的单图输出使用 `dim=0`；对 `[batch, 10]` 使用 `dim=1`，分别在每张图片的类别上归一化。对后一种形状误用 `dim=0` 会跨图片归一化。Softmax 不改变同一行最大分数的位置，分类可直接使用 `logits.argmax(dim=1)`；归一化分布也不自动等于经过校准的真实正确概率。[^原一]

## CrossEntropyLoss、梯度与优化器

对于整数类别标签，交叉熵可以写成正确类别归一化分数的负对数：

$$
L=-\log p_y
$$

PyTorch 的 `CrossEntropyLoss` 接收原始 Logits，在内部完成与 LogSoftmax 和负对数似然等价的计算；先手工 Softmax 再传入会改变这个训练问题。脚本每批使用默认平均损失。[^接口]

训练顺序是清空旧梯度 → 前向计算 → 计算损失 → `loss.backward()` → `optimizer.step()`。不清空会累积梯度；本例不做梯度累积，所以每批清零。手写更新时可在 `torch.no_grad()` 内执行 `param -= learning_rate * param.grad`，但要先判断 `param.grad is not None`：冻结或未参与当前计算图的参数可以没有梯度，这与“梯度张量恰好为零”不同。Adam 代替手写更新，学习率取原示例的0.01。[^优化]

训练模式用 `model.train()`；测试用 `model.eval()` 和 `torch.no_grad()`。`eval()` 改变部分层的训练行为，`no_grad()` 关闭梯度记录，二者不能互相替代。虽然本网络没有 Dropout、BatchNorm，仍保留明确的训练与测试边界。

## 完整可运行脚本

本次环境：Python **3.14.4**、PyTorch **2.14.0**、torchvision **0.29.0**，macOS 26.4.1 arm64，**CPU、float32、4线程、随机种子42**。种子在模型初始化和 DataLoader 随机采样之前设置，训练启用确定性算法检查。不同版本、设备及底层库仍不能保证得到相同结果。

使用 Python 3.14.4，把代码保存为 `mnist.py` 后运行：

```sh
python3 -m venv .venv
.venv/bin/python -m pip install 'torch==2.14.0' 'torchvision==0.29.0'
.venv/bin/python mnist.py --device cpu --output ./mnist-run
```

首次运行会下载 MNIST；后续复用输出目录中的数据。本文配套保存了实际命令、完整环境、运行日志、结果JSON与权重。若系统缺少可信根证书，应正确配置证书链，不禁用 TLS 校验。本次使用已安装 `certifi` 提供的可信根完成下载，确切命令保存在执行记录。

`--device mps` 仅在 `torch.backends.mps.is_available()` 为真时允许继续；还需要当前算子满足确定性算法要求，否则运行会失败并需如实处理。**本文没有执行 MPS 训练，以下结果全部来自 CPU。**

```python
"""运行完整 MNIST 训练，并验证参数更新、测试损失及保存加载一致性。"""
import argparse
import hashlib
import importlib.metadata
import json
import math
import platform
import random
from pathlib import Path

import torch
from torch import nn
from torch.utils.data import DataLoader
from torchvision.datasets import MNIST
from torchvision.transforms.functional import to_tensor


def img_preprocess(image):
    return to_tensor(image).flatten()


class MnistModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(784, 256), nn.ReLU(),
            nn.Linear(256, 128), nn.ReLU(), nn.Linear(128, 10),
        )

    def forward(self, x):
        return self.net(x)


def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument('--device', choices=['cpu', 'mps'], default='cpu')
    parser.add_argument('--output', type=Path, required=True)
    args = parser.parse_args()
    args.output.mkdir(parents=True, exist_ok=True)
    if args.device == 'mps' and not torch.backends.mps.is_available():
        raise RuntimeError('当前环境不支持 MPS，不能将 CPU 结果标为 MPS 实测')
    seed = 42
    random.seed(seed)
    torch.manual_seed(seed)
    torch.set_num_threads(4)
    torch.use_deterministic_algorithms(True)
    device = torch.device(args.device)
    environment = {
        'Python': platform.python_version(), '系统': platform.platform(),
        'PyTorch': torch.__version__,
        'torchvision': importlib.metadata.version('torchvision'),
        '设备': str(device), '精度': 'float32', '随机种子': seed,
        '训练批量': 50, '测试批量': 1000, '学习率': 0.01,
        '优化器': 'Adam', '轮数': 5, 'CPU线程': torch.get_num_threads(),
    }
    print(json.dumps(environment, ensure_ascii=False), flush=True)
    train_data = MNIST(args.output / 'data', train=True, download=True, transform=img_preprocess)
    test_data = MNIST(args.output / 'data', train=False, download=True, transform=img_preprocess)
    assert len(train_data) == 60000 and len(test_data) == 10000
    assert train_data.data.dtype == torch.uint8
    assert tuple(train_data.data.shape) == (60000, 28, 28)
    assert int(train_data.data.min()) == 0 and int(train_data.data.max()) == 255
    train_loader = DataLoader(train_data, batch_size=50, shuffle=True,
                              generator=torch.Generator().manual_seed(seed), num_workers=0)
    test_loader = DataLoader(test_data, batch_size=1000, shuffle=False, num_workers=0)
    model = MnistModel().to(device)
    before = {name: value.detach().clone() for name, value in model.named_parameters()}
    criterion = nn.CrossEntropyLoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=0.01)
    epochs = []
    for epoch in range(1, 6):
        model.train()
        total_loss, count = 0.0, 0
        for images, labels in train_loader:
            images, labels = images.to(device), labels.to(device)
            assert images.dtype == torch.float32 and images.shape[1] == 784
            optimizer.zero_grad(set_to_none=True)
            logits = model(images)
            assert logits.shape == (labels.shape[0], 10)
            loss = criterion(logits, labels)
            assert torch.isfinite(loss).item(), '训练损失必须有限'
            loss.backward()
            assert all(p.grad is None or torch.isfinite(p.grad).all().item() for p in model.parameters())
            optimizer.step()
            total_loss += loss.item() * labels.numel()
            count += labels.numel()
        assert count == 60000
        entry = {'轮次': epoch, '训练样本数': count, '训练平均损失': total_loss / count}
        epochs.append(entry)
        print(json.dumps(entry, ensure_ascii=False), flush=True)
    changes = {name: (value.detach() - before[name]).abs().max().item()
               for name, value in model.named_parameters()}
    assert all(math.isfinite(value) and value > 0 for value in changes.values())
    model.eval()
    total_loss, correct, count = 0.0, 0, 0
    with torch.no_grad():
        for images, labels in test_loader:
            images, labels = images.to(device), labels.to(device)
            logits = model(images)
            loss = criterion(logits, labels)
            assert torch.isfinite(logits).all().item() and torch.isfinite(loss).item()
            total_loss += loss.item() * labels.numel()
            correct += (logits.argmax(dim=1) == labels).sum().item()
            count += labels.numel()
        assert count == 10000
        fixed_images, _ = next(iter(test_loader))
        fixed_images = fixed_images.to(device)
        saved_logits = model(fixed_images)
        weights = args.output / 'output.pth'
        torch.save(model.state_dict(), weights)
        loaded = MnistModel().to(device)
        loaded.load_state_dict(torch.load(weights, map_location=device, weights_only=True))
        loaded.eval()
        loaded_logits = loaded(fixed_images)
        same_classes = torch.equal(saved_logits.argmax(1), loaded_logits.argmax(1))
        max_error = (saved_logits - loaded_logits).abs().max().item()
        assert same_classes
        assert torch.allclose(saved_logits, loaded_logits, rtol=0.0, atol=1e-6)
    result = {
        '环境': environment, '训练记录': epochs, '参数最大变化': changes,
        '测试样本数': count, '测试平均损失': total_loss / count,
        '测试正确数': correct, '测试准确率': correct / count,
        '保存加载': {'固定批次样本数': len(fixed_images), '类别完全一致': same_classes,
                     '输出最大绝对误差': max_error, 'rtol': 0.0, 'atol': 1e-6},
        '权重SHA256': hashlib.sha256(weights.read_bytes()).hexdigest(),
    }
    (args.output / '结果.json').write_text(json.dumps(result, ensure_ascii=False, indent=2) + '\n')
    print(json.dumps(result, ensure_ascii=False), flush=True)
    print('全部检查通过', flush=True)


if __name__ == '__main__':
    main()
```

## 独立测试与保存加载结果

每轮都遍历60,000张训练图，先检查每批损失和梯度有限；训练完成后确认每个参数组相对初始化都有有限且非零的变化。

| Epoch | 训练样本数 | 按样本加权的平均损失 |
| --- | --- | --- |
| 1 | 60000 | 0.2563031452 |
| 2 | 60000 | 0.1611299682 |
| 3 | 60000 | 0.1332766082 |
| 4 | 60000 | 0.1281564930 |
| 5 | 60000 | 0.1133692933 |

平均损失按 `sum(批平均损失 × 批样本数) / 总样本数` 计算，避免最后一个小批次与完整批次被等权平均。测试单独遍历全部10,000张图，没有用训练样本或单条预测代替：

- 测试平均损失：**0.14691142849624156**。
- 正确识别：**9635 / 10000**，准确率 **96.35%**。
- 保存后新建同架构模型，使用 `weights_only=True` 和同设备 `map_location` 加载；固定同一批1000张测试图，在相同float32、评估模式下比较。
- 加载前后预测类别完全一致，Logits最大绝对误差 **0.0**，验收容差为 `rtol=0.0`、`atol=1e-6`。[^实测]

保存的是 `state_dict`；`weights_only=True` 限制反序列化内容，不意味着任意陌生权重都可信。本文只重新加载自身本次生成的文件。原资料缺少可复核的预训练权重，因此不沿用索引88图片的0.9999概率；独立测试结果以上述脚本为准。

## 来源与版本

| 编号 | 准确标题 | 作者/机构 | 发布日期/版本 | URL或本地原件 | 定位 | 支持范围 |
| --- | --- | --- | --- | --- | --- | --- |
| 原一 | 从零搭建神经网络，识别手写数字【PyTorch】【Transformer结构拆解】 | 隔壁的程序员老王 | 2025-11-27（原稿属性） | [视频](https://www.bilibili.com/video/BV1ypUkB7Eki) | Linear、张量、展平、Logits、Softmax | 原示例的模型结构与推理解释 |
| 原二 | 30行代码训练模型 手写数字识别【PyTorch实战】 | 隔壁的程序员老王 | 2025-12-25（原稿属性） | [视频](https://www.bilibili.com/video/BV1SdBcB7EtG) | Dataset、DataLoader、损失、完整训练循环、Optimizer | 批量50、五轮、学习率0.01及训练流程 |
| 数据 | torchvision.datasets.MNIST；to_tensor | PyTorch项目 | torchvision 0.29.0 | 已安装版本接口 | MNIST、MNIST.__getitem__、to_tensor | 下载、PIL转换、像素缩放；数量与类型由正文程序检查 |
| 接口 | Linear；CrossEntropyLoss；Tensor.view；torch.load | PyTorch项目 | PyTorch 2.14.0 | 已安装版本接口 | 各同名对象 | 最后一维、Logits输入、步长约束、参数加载 |
| 优化 | Optimizing Model Parameters | PyTorch项目 | 2026-09-06读取 | [官方教程](https://docs.pytorch.org/tutorials/beginner/basics/optimization_tutorial.html) | Full Implementation、Optimization Loop | 训练测试模式、梯度与优化顺序；官方示例为FashionMNIST，不能证明本文MNIST数值 |
| 实测 | 本文正文中的MNIST完整检查程序 | 本文所列环境 | 2026-09-06运行 | 正文完整代码及文中结果 | main；环境、训练、测试及保存加载检查 | 本文CPU结果及验收断言 |

[^原一]: 来源表“原一”；保留结构、张量及Softmax维度说明，不沿用缺权重的单例数值。
[^数据]: 来源表“数据”及脚本对MNIST数量、形状、范围的断言。
[^接口]: 来源表“接口”；快照从本次实际安装的PyTorch 2.14.0提取。
[^优化]: 来源表“原二”“优化”；梯度为None另见API快照Optimizer.zero_grad说明。
[^实测]: 来源表“实测”；测试与保存加载结果来自实际日志，未把CPU结果标作MPS。
