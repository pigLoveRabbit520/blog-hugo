---
title: pytorch学习哈
author: pigLoveRabbit
tags:
  - pytorch
categories:
  - 深度学习
date: 2024-12-30 22:00:00
---
![pytorch](/images/PyTorchLogo.jpg)
<!-- more -->

## backward函数
`backward()` 是深度学习框架（如 **PyTorch**）中的一个核心方法，它的作用是：**自动计算损失函数对模型参数的梯度（gradient）**，为后续的参数更新（比如用 SGD、Adam 等优化器）提供依据。

---

### ✅ 一句话概括：
> `loss.backward()` 会**从损失标量开始，反向传播**，把梯度自动存到每个 `requires_grad=True` 的张量的 `.grad` 属性中。

---

### 🧠 举个例子（PyTorch 风格）

```python
import torch

# 定义参数（需要梯度）
w = torch.tensor([2.0], requires_grad=True)  # 标量参数，也可以是向量

# 输入和标签
x = torch.tensor([3.0])
y_true = torch.tensor([7.0])

# 前向计算：预测 = w * x
y_pred = w * x

# 计算损失（标量）
loss = (y_pred - y_true) ** 2  # 比如 MSE，这里没除2

print("loss:", loss)  # loss = (2*3 - 7)^2 = (-1)^2 = 1

# 反向传播！
loss.backward()

print("w.grad:", w.grad)  # 输出：tensor([-6.])
```

#### 🔍 手动验证梯度是否正确？
损失函数：  
\[
\mathcal{L} = (w x - y)^2
\]  
对 \(w\) 求导：  
\[
\frac{\partial \mathcal{L}}{\partial w} = 2(w x - y) \cdot x = 2(6 - 7) \cdot 3 = -6
\]  
✅ 和 `w.grad` 一致！

---

### 🔄 `backward()` 背后做了什么？

1. **构建计算图**（动态图）：  
   PyTorch 在前向计算时，会悄悄记录所有操作（如 `*`, `+`, `**2`），形成一个**有向无环图**（DAG）。

2. **从 loss 开始反向遍历图**：  
   利用链式法则（chain rule），逐层计算每个中间变量、每个参数的梯度。

3. **累加梯度到 `.grad`**：  
   如果多次调用 `backward()`，梯度会**累加**（所以训练循环中通常要 `optimizer.zero_grad()` 清零）。

---

### ⚠️ 注意事项

| 问题 | 说明 |
|------|------|
| **只能对标量调用 `backward()`** | 如果 `loss` 是向量，需指定 `grad_tensors`，或先用 `.sum()`/`.mean()` 变成标量 |
| **梯度会累加** | 每次 `backward()` 不会覆盖 `.grad`，而是 `+=`，所以训练时要清零 |
| **只对 `requires_grad=True` 的张量计算梯度** | 默认 `torch.tensor(...)` 的 `requires_grad=False` |

---

### 🧩 在训练循环中的典型用法

```python
for data, target in dataloader:
    optimizer.zero_grad()        # 👈 清零上一步的梯度
    output = model(data)
    loss = criterion(output, target)
    loss.backward()              # 👈 反向传播：计算 d(loss)/d(params)
    optimizer.step()             # 👈 用梯度更新参数（如 w -= lr * w.grad）
```

---

### 总结

| 概念 | 说明 |
|------|------|
| `backward()` | 自动求导，计算损失对所有可学习参数的梯度 |
| 结果存储在哪？ | `param.grad`（和 `param` 同 shape） |
| 为什么重要？ | 没有它，就无法用梯度下降更新模型参数 |
| 底层原理？ | 动态计算图 + 链式法则 + 反向传播 |

## @ 运算符
实际上，`@` 运算符在 PyTorch 中就是 torch.matmul 的语法糖。
```
w = torch.randn(12, 1)   # 列向量 (12, 1)
x = torch.randn(12, 1)   # 列向量 (12, 1)
result = w.t() @ x     # (1, 1) 标量，
print(result)
```

### 底层对应关系

| 表达式 | 等价函数 |
|--------|--------|
| `a @ b`（1D @ 1D） | `torch.dot(a, b)` |
| `a @ b`（1D @ 2D） | `torch.mv(b.t(), a).t()` 或自动处理 |
| `a @ b`（2D @ 2D） | `torch.mm(a, b)` |
| `a @ b`（3D @ 3D） | `torch.bmm(a, b)` |
| `a @ b`（高维） | `torch.matmul(a, b)` |
