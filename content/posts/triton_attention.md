---
title: Triton写Attention算子
author: pigLoveRabbit
tags: []
categories:
  - python
  - Triton
date: 2025-10-26 14:00:00
---
![upload successful](/images/triton-logo.png)  


## Triton
[官网介绍](https://triton.hyper.ai/)：Triton 是一种用于并行编程的语言和编译器。它旨在提供一个基于 Python 的编程环境，以高效编写自定义 DNN 计算内核，并能够在现代 GPU 硬件上以最大吞吐量运行。  
简单讲，就是可以用Python写GPU算子。  
<!-- more -->



## 最简单的Attention算子

Attention公式
$$
\text{Attention}(Q, K, V) = \text{softmax}\left( \frac{Q K^\top}{\sqrt{d_k}} \right) V
$$

```python
import torch
import torch.nn.functional as F

def simple_attention(query, key, value):
    """
    最简单的 scaled dot-product attention
    Args:
        query: (..., seq_len_q, d_k)
        key:   (..., seq_len_k, d_k)
        value: (..., seq_len_k, d_v)
    Returns:
        output: (..., seq_len_q, d_v)
        attention_weights: (..., seq_len_q, seq_len_k)
    """
    d_k = query.size(-1)
    scores = torch.matmul(query, key.transpose(-2, -1)) / torch.sqrt(torch.tensor(d_k, dtype=torch.float32))
    attention_weights = F.softmax(scores, dim=-1)
    output = torch.matmul(attention_weights, value)
    return output, attention_weights

# 示例用法
if __name__ == "__main__":
    # 假设 batch_size=2, seq_len=3, d_k=d_v=4
    batch_size, seq_len, d_k, d_v = 2, 3, 4, 4
    Q = torch.randn(batch_size, seq_len, d_k)
    K = torch.randn(batch_size, seq_len, d_k)
    V = torch.randn(batch_size, seq_len, d_v)

    output, weights = simple_attention(Q, K, V)

    print("Output shape:", output.shape)        # [2, 3, 4]
    print("Weights shape:", weights.shape)      # [2, 3, 3]
```
这个实现没有 mask、没有多头、没有线性变换，是最核心、最简化的 attention 逻辑。适合学习理解。

### 关键部分解释
1. `query`、`key`、`value` 是标准的注意力三元组。
2. 计算 query 和 key 的点积（相似度）。
3. 除以 sqrt(d_k) 进行缩放（防止点积过大导致 softmax 梯度消失）。
4. 对结果做 softmax 得到注意力权重。
5. 用权重对 value 加权求和，得到输出。

## Transfomer模型
![transformer](https://transformers.run/assets/img/attention/transformer.jpeg)
左边是Encoder，右边是Decoder  

**在推理阶段，没有“Outputs (shifted right)”这个明确的输入。** 这个概念是训练阶段为了模拟推理行为而设计的一种“技巧”或“教学工具”。

我们来详细解释一下：

---

### 1. 训练阶段的 “Outputs (shifted right)” 是什么？为什么需要它？

在训练时，我们为了并行高效，会一次性把整个目标序列（比如 “我 爱 你 <eos>”）喂给Decoder。

*   **问题：** 如果直接输入 “我 爱 你 <eos>”，在预测第一个词 “我” 的时候，模型就会“偷看”到后面的 “爱”、“你”，这违反了“根据上文预测下一个词”的自回归法则。这被称为**信息泄露**。

*   **解决方案：** “Shifted Right”（右移）
    *   **Decoder的输入：** 我们把目标序列的**开始符 `<sos>`** 放在最前面，并**去掉最后一个词**，变成：`<sos> 我 爱`
    *   **训练标签/目标：** 就是原始的目标序列，但**去掉开始符**，变成：`我 爱 你 <eos>`

这样，当Decoder的输入是 `<sos> 我 爱` 时，它被要求并行地输出 `我 爱 你 <eos>`。我们来看一下这个对应关系：

| 输入 (Shifted Right) | → | 模型被要求预测的输出 |
| :--- | :--- | :--- |
| `<sos>` | → | **我** |
| `<sos>` **我** | → | **爱** |
| `<sos>` 我 **爱** | → | **你** |
| `<sos>` 我 爱 **你** | → | **`<eos>`** |

通过**掩码自注意力**，确保在计算每个位置的输出时，只能看到它自己和它左边的输入。这样，对于“预测‘爱’”这个任务，模型的输入能看到 `<sos>` 和 “我”，但看不到后面的“爱”和“你”，完美模拟了推理时的情况。

**所以，“Outputs (shifted right)”是训练时为了并行化而构造的一个“伪输入”。**

---

### 2. 推理阶段的输入是什么？

推理时，我们**没有**完整的目标序列，因此无法构造一个“shifted right”的版本。推理的输入是**动态生成**的。

**推理输入 = 起始符 + 模型自己历史生成的全部令牌**

这个过程是循环进行的：
1.  **初始输入：** `[<sos>]`
2.  **第一轮输出：** 模型接收 `[<sos>]`，输出第一个词 `我`。
3.  **更新输入：** 将 `我` 追加到输入中，得到 `[<sos>, 我]`
4.  **第二轮输出：** 模型接收 `[<sos>, 我]`，输出下一个词 `爱`。
5.  **更新输入：** 将 `爱` 追加到输入中，得到 `[<sos>, 我, 爱]`
6.  ... 如此循环，直到生成 `<eos>`。

### 总结对比

| 阶段 | 输入来源 | “Outputs (shifted right)”？ |
| :--- | :--- | :--- |
| **训练** | 来自**训练数据集**中准备好的完整目标序列。 | **有**。这是一个为了并行训练而特意构造的输入。 |
| **推理** | 来自**模型自身前一步的输出**，动态拼接而成。 | **没有**。因为根本没有现成的“Outputs”可供“Shift”。 |

**一个形象的比喻：**

*   **训练：** 像老师在教学生完形填空。老师直接把一整段文字（shifted right）给学生看，但用纸盖住后面的部分（掩码），让学生同时填写所有的空。
*   **推理：** 像学生自己写作文。他只能从第一个字开始写，写下的每一个字都成为下文的基础。他不可能提前拿到一篇“右移”的作文。

因此，**“Outputs (shifted right)”是训练阶段独有的一个数据准备步骤，在推理阶段不存在。** 推理阶段的输入是自回归地、逐步构建起来的。

##是的，**Transformer 中的 FFN（Feed-Forward Network，前馈神经网络）层本质上是由两个全连接层（Fully Connected Layers）组成的**，中间夹着一个非线性激活函数。

### FFN层：
在原始的 Transformer 论文（《Attention is All You Need》）中，FFN 层的结构如下：

```
FFN(x) = W₂ * GELU(W₁ * x + b₁) + b₂
```

或者在早期版本中使用的是 ReLU：

```
FFN(x) = W₂ * ReLU(W₁ * x + b₁) + b₂
```

其中：
- `W₁` 和 `W₂` 是可学习的权重矩阵（即全连接层的参数），
- `b₁` 和 `b₂` 是偏置项，
- `GELU` 或 `ReLU` 是非线性激活函数。

### 为什么说它们是“全连接”？
- **第一层**：将输入向量（维度为 `d_model`）线性变换到一个更高维的隐藏空间（维度通常为 `4 * d_model`），这是一个标准的线性变换 → **全连接层**。
- **激活函数**：引入非线性（如 ReLU、GELU）。
- **第二层**：将隐藏层向量再线性变换回原始维度 `d_model` → **另一个全连接层**。

这两个线性变换都没有使用卷积、循环结构或注意力机制，纯粹是矩阵乘法 + 偏置，因此**完全符合“全连接层”的定义**。

#### 补充说明：
- **每个位置独立处理**：FFN 对序列中每个位置的表示独立应用相同的网络结构（即权重共享），这与自注意力机制不同（自注意力是跨位置交互的）。
- **参数量大**：由于隐藏层维度通常是 `d_model` 的 4 倍，FFN 是 Transformer 中参数量最大的部分之一（通常超过注意力机制）。
- **变体存在**：虽然标准 FFN 是两个全连接层，但后续研究也探索了其他结构（如使用卷积、稀疏连接、MoE 等），但在原始 Transformer 和绝大多数实现中，**FFN 就是两个全连接层**。

### 总结：
✅ **是的，Transformer 中的 FFN 层由两个全连接层构成**，中间加一个非线性激活函数。这是其标准和最广泛使用的结构。

在你提到的 Transformer 架构图（例如 [https://transformers.run/assets/img/attention/transformer.jpeg](https://transformers.run/assets/img/attention/transformer.jpeg)）中，**“Add & Norm”** 是 **“Add and Layer Normalization”** 的缩写，它是 Transformer 模型中一个**关键的子层结构**，用于稳定训练、加速收敛并提升模型性能。

---

### Add & Norm 一句话解释：
> **“Add & Norm” = 残差连接（Add） + 层归一化（Layer Normalization）**

它出现在 **Encoder 和 Decoder 的每个子模块之后**，具体有两个位置：
1. **自注意力（或 cross-attention）之后**
2. **前馈神经网络（Feed Forward）之后**

---

### 🔍 详细拆解：

#### 1. **Add（残差连接 / Residual Connection）**
- 将**子层的输入**和**子层的输出**相加：
  ```python
  output = sublayer(input) + input
  ```
- 作用：
  - 缓解深层网络中的梯度消失问题；
  - 允许信息直接“跳过”某些层，保留原始特征；
  - 是 ResNet 思想在 Transformer 中的应用。

#### 2. **Norm（Layer Normalization）**
- 对 **Add 之后的结果**做层归一化：
  ```python
  normalized_output = LayerNorm(sublayer(input) + input)
  ```
- LayerNorm 是对**单个样本的所有特征维度**做归一化（与 BatchNorm 不同，不依赖 batch）；
- 作用：
  - 稳定隐藏层分布；
  - 加速训练；
  - 减少对初始化和学习率的敏感性。

---

### 📦 在 Transformer 中的具体位置：

#### Encoder 层结构（重复 N 次）：
```
Input
  │
  ├─→ [Multi-Head Self-Attention] ──→ Add & Norm ──→
  │
  └─→ [Feed Forward Network] ───────→ Add & Norm ──→ Output
```

#### Decoder 层结构（重复 N 次）：
```
Input
  │
  ├─→ [Masked Multi-Head Self-Attention] ──→ Add & Norm ──→
  │
  ├─→ [Multi-Head Cross-Attention] ─────────→ Add & Norm ──→
  │
  └─→ [Feed Forward Network] ───────────────→ Add & Norm ──→ Output
```

> ⚠️ 注意：原始论文中是 **先做子层计算 → 再 Add → 再 Norm**（Post-LN）。  
> 但现在很多变体（如 Pre-LN）会把 LayerNorm 放在子层**之前**，训练更稳定。

---

### 🧠 举个简单例子（数值示意）：

假设某层输入是向量 `x = [1.0, 2.0]`，经过自注意力后输出 `y = [0.8, 2.2]`。

1. **Add（残差）**：  
   `z = x + y = [1.0+0.8, 2.0+2.2] = [1.8, 4.2]`

2. **Norm（LayerNorm）**：  
   对 `[1.8, 4.2]` 做归一化（减均值、除标准差 + 可学习缩放/偏移），得到标准化后的向量，比如 `[−1, 1]`（简化示意）。

这个结果就作为下一层的输入。

---

### ❓为什么叫 “Add & Norm” 而不是 “Norm & Add”？
- 因为**顺序是先 Add，再 Norm**（在原始 Transformer 中）；
- 虽然名字是 “Add & Norm”，但实际是两个操作的组合，不是并行的。

---

### ✅ 总结：

| 组件 | 作用 |
|------|------|
| **Add** | 残差连接，保留原始信息，缓解梯度消失 |
| **Norm** | 层归一化，稳定训练，加速收敛 |
| **Add & Norm** | Transformer 的标准子层后处理模块，提升模型深度和性能 |

它是 Transformer 能堆叠多层（如 6 层、12 层甚至更多）而依然可训练的关键设计之一！

如果你看到图中每个方框（Attention / FFN）后面都跟着一个 “Add & Norm”，现在你就知道它在做什么啦 😊


## 参考
* [Transformer 模型](https://transformers.run/c1/transformer/#transformer-%E7%9A%84%E7%BB%93%E6%9E%84)
