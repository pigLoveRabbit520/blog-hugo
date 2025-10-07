---
title: pytorch手写数字识别
author: pigLoveRabbit
tags:
  - pytorch
categories:
  - 深度学习
date: 2024-10-29 20:00:00
---
![pytorch](/images/PyTorchLogo.jpg)
<!-- more -->

## 一个例子
csv在这里[下载](https://github.com/npradaschnor/Pima-Indians-Diabetes-Dataset/blob/master/diabetes.csv)
```Python
import torch
import torchvision
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms

# 数据预处理
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.5,), (0.5,))
])

# 加载数据集
train_dataset = datasets.MNIST('data/', train=True, download=True, transform=transform)
test_dataset = datasets.MNIST('data/', train=False, download=True, transform=transform)

train_loader = torch.utils.data.DataLoader(train_dataset, batch_size=64, shuffle=True)
test_loader = torch.utils.data.DataLoader(test_dataset, batch_size=64, shuffle=False)

# 定义神经网络模型
class NeuralNetwork(nn.Module):
    def __init__(self):
        super(NeuralNetwork, self).__init__()
        self.layer1 = nn.Linear(784, 128)
        self.layer2 = nn.Linear(128, 64)
        self.layer3 = nn.Linear(64, 10)

    def forward(self, x):
        x = torch.relu(self.layer1(x))
        x = torch.relu(self.layer2(x))
        x = self.layer3(x)
        return x

# 实例化模型
model = NeuralNetwork()

# 定义损失函数和优化器
loss_func = nn.CrossEntropyLoss()
optimizer = optim.SGD(model.parameters(), lr=0.01)

# 训练模型
epochs = 5
for epoch in range(epochs):
    for batch, (images, labels) in enumerate(train_loader):
        optimizer.zero_grad()
        outputs = model(images.view(-1, 784))
        loss = loss_func(outputs, labels)
        loss.backward()
        optimizer.step()

# 在测试集上评估模型
with torch.no_grad():
    correct = 0
    total = 0
    for images, labels in test_loader:
        outputs = model(images.view(-1, 784))
        _, predicted = torch.max(outputs, 1)
        total += labels.size(0)
        correct += (predicted == labels).sum().item()
    accuracy = correct / total
    print(f"Accuracy: {accuracy}")
```

以上代码分为几个过程
1. 数据预处理     - 定义了数据的转换操作，包括将图像转换为张量并进行标准化。  
2. 数据加载     - 加载了 MNIST 数据集的训练集和测试集，并创建了数据加载器。  
3. 模型定义     - 构建了一个包含三层线性层的神经网络模型。  
4. 损失函数与优化器定义     - 选择了交叉熵损失函数，并使用随机梯度下降（SGD）作为优化器。  
5. 模型训练     - 通过多个轮次（epochs）的迭代，对模型进行训练。在每个批次的数据中，计算损失，进行反向传播和参数更新。  
6. 模型评估     - 在测试集上对训练好的模型进行评估，计算预测正确的数量并得出准确率。