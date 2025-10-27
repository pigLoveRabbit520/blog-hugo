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


