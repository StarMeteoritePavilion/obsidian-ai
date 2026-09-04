---
title: 30行代码训练模型 手写数字识别【PyTorch实战】
source: https://www.bilibili.com/video/BV1SdBcB7EtG
author: 隔壁的程序员老王
created: 2025-12-25
tags:
  - AI
  - 模型训练
  - PyTorch
  - MNIST
  - CrossEntropyLoss
---

> “从0开始一起学大模型”合集第五条／PyTorch 手写数字识别训练实战

手写数字识别模型的输入是经过预处理的 MNIST 图片，输出是数字 0 至 9 对应的十个 Logits。要让随机初始化的模型完成识别，还需要把数据预处理、批量加载、损失计算、梯度更新和参数保存连成一条完整训练链路。

## 回顾输入与模型

MNIST 的原始图片是 $28\times28$ 的二维张量，每个元素都是 0 至 255 的整数，代表对应像素的灰度值。把每行像素首尾相连，可以将一张图片展平为形状为 $784$ 的张量；再除以 255，便可把数值缩放到 0 至 1。

经过这两步处理，60000 张训练图片组成形状为 $60000\times784$ 的张量，10000 张测试图片组成形状为 $10000\times784$ 的张量。识别模型依次使用三个 Linear 和两个 ReLU，把 784 个输入值映射为十个 Logits：

$$
784\rightarrow256\rightarrow128\rightarrow10
$$

这十个数分别表示输入图片与数字 0 至 9 的匹配程度。

## 用 Dataset 按需处理数据

直接通过 MNIST 的 `data` 属性读取全部数据，再调用 `view()`、`float()` 和除以 255 完成预处理，要求先把所有数据加载到内存。本例的数据集只有四十多 MB，可以采用这种方式；实际工程中的数据可能达到 TB 级，全部载入内存并不现实。

PyTorch 在文件与内存之间提供了 Dataset 抽象。程序可以像访问数组一样读取其中任意元素，Dataset 负责根据访问需求提供数据。MNIST 本身就是一个 Dataset，直接修改其内部 `data` 属性并不是资料建议的常规做法。

更合适的方式是在创建 MNIST 实例时传入 `transform` 函数。Dataset 每次返回数据时调用该函数，把预处理放在单个样本的读取过程中。`torchvision` 会先把 MNIST 的原始张量转换为 `PIL.Image.Image`；这种形式便于执行缩放、裁剪等图像处理。

本例使用 `torchvision.transforms.functional.to_tensor()` 把 Image 转回张量，同时将像素值缩放到 0 至 1，再用 `view(28 * 28)` 展平：

```python
from torch import nn, Tensor
from PIL.Image import Image
from torchvision.transforms import functional


def img_preprocess(img: Image) -> Tensor:
    tensor = functional.to_tensor(img)
    tensor = tensor.view(28 * 28)
    return tensor
```

处理后的 Dataset 每次返回一个 Tuple：第一个元素是形状为 $784$、取值为 0 至 1 的图片张量，第二个元素是图片所代表的数字。示例读取第 90 条数据，标签为 6。显示图片时则继续使用未经预处理的 `data` 属性。

## 用 DataLoader 组成随机批次

本例把 Batch Size 设置为 50，即每次使用 50 张图片训练。Batch Size 是需要人工设置的超参数。

`DataLoader` 接收 Dataset 作为数据源，并通过 `batch_size=50` 指定批量大小。把 `shuffle` 设为 `True` 后，数据顺序会被打乱。它可以直接用于 `for` 循环，每次返回一批预处理图片及其标签：图片张量的形状为 $50\times784$，标签张量的形状为 $50$。

```python
from torch.utils.data import DataLoader

dataloader = DataLoader(
    dataset=mnist,
    batch_size=50,
    shuffle=True,
)
```

## CrossEntropyLoss 从 Logits 计算分类损失

模型输出的 Logits 只是十个分数。Softmax 可以把它们转换为概率；如果图片的正确标签是 6，理想输出就是数字 6 的概率接近 1，其余类别的概率接近 0。因为全部概率之和为 1，只要正确标签的概率升高，其他类别的概率就会相应降低。

本例需要的损失函数满足：正确标签的概率越接近 1，损失越接近 0；正确标签的概率越小，损失越大。CrossEntropyLoss 使用的对应关系可以写为：

$$
Loss=-\ln(p)
$$

其中，$p$ 是正确标签的概率。$p$ 越接近 1，损失变化越慢；$p$ 越接近 0，损失变化越快。资料据此说明，预测接近正确结果时参数调整幅度应减小，偏离较远时调整幅度应增大。Softmax 中的指数运算与对数也为内部实现留下了优化空间。

PyTorch 的 `CrossEntropyLoss` 已经包含 Softmax 对应的处理，因此模型输出层不要再次添加 Softmax，直接输出 Logits。损失函数接收模型输出和正确标签；例如两张图片的输出形状为 $2\times10$，标签分别是 0 和 2，最终会得到一个损失值。

```python
criterion = torch.nn.CrossEntropyLoss()
```

## 完整训练循环

训练包含两层循环。内层循环遍历 DataLoader，每次取得 50 张图片，直到用完训练集；外层循环把整个训练集重复训练五次。完整遍历一次训练集称为一个 Epoch，本例的 `epoch=5` 也是超参数。

每个批次依次执行四步：

1. 把图片送入模型，得到 Logits。
2. 用 `CrossEntropyLoss` 根据 Logits 和 Labels 计算损失。
3. 清空上一轮梯度，再调用 `loss.backward()` 自动计算本轮梯度。
4. 根据梯度与 Learning Rate 更新参数。

```python
mnist_model = MnistModel()
learning_rate = 0.01
epoch = 5

for _ in range(epoch):
    for images, labels in dataloader:
        logits = mnist_model(images)
        loss = criterion(logits, labels)

        mnist_model.zero_grad()
        loss.backward()

        with torch.no_grad():
            for param in mnist_model.parameters():
                if param.grad is not None:
                    param -= param.grad * learning_rate
```

计算新一轮梯度前必须调用 `zero_grad()`，否则上一轮梯度会继续保留。手动更新参数的代码放在 `torch.no_grad()` 上下文管理器中，使这项操作不进入下一轮梯度计算。

参数梯度还可能是 `None`：一种情况是训练者有意冻结某些参数，例如 LoRA 可以锁住部分参数；另一种情况是某个参数在当前数据下没有对损失产生贡献。因此，手动更新前需要检查 `param.grad is not None`。

## 验证与保存参数

完成五个 Epoch 后，示例取出训练集中的第 90 条数据。模型输出的十个 Logits 中，数字 6 对应的值最大，与图片标签一致。

这次操作直接使用了训练集数据，只用于展示训练结果。实际工程需要使用测试集验证模型，以检查模型对未参与训练的数据是否仍然有效，并防止只根据训练集结果判断模型表现。

训练好的参数可以通过 `torch.save()` 保存，再由模型的 `load_state_dict()` 加载；加载时在 `torch.load()` 中使用 `weights_only=True`：

```python
torch.save(mnist_model.state_dict(), "output.pth")

mnist_model.load_state_dict(
    torch.load("output.pth", weights_only=True)
)
```

## 用 Optimizer 代替手动更新

前面的训练循环使用固定 Learning Rate 0.01，并手动逐个更新参数。资料指出，训练早期通常希望更快降低损失，训练后期则需要避免步幅过大而难以收敛；实际工程通常使用 PyTorch 已经实现的 Optimizer，而不是手写参数更新逻辑。

以 Adam 为例，创建优化器时传入需要调整的全部模型参数和初始 Learning Rate。在训练循环中，由 `optimizer.zero_grad()` 清空梯度，再通过 `optimizer.step()` 根据算法更新参数：

```python
optimizer = torch.optim.Adam(
    mnist_model.parameters(),
    lr=learning_rate,
)

for _ in range(epoch):
    for images, labels in dataloader:
        logits = mnist_model(images)
        loss = criterion(logits, labels)

        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

至此，数据按批次进入模型，CrossEntropyLoss 把分类结果转换为训练信号，PyTorch 自动计算梯度，Optimizer 完成参数更新，训练后的权重则可以保存并重新加载。模型结构、训练数据、损失函数和参数更新由此组成一条完整训练链路。
