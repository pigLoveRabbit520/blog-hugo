---
title: Triton写简单算子
author: pigLoveRabbit
tags: []
categories:
  - python
  - Triton
date: 2025-10-02 14:00:00
---
![upload successful](/images/triton-logo.png)  


## Triton
[官网介绍](https://triton.hyper.ai/)：Triton 是一种用于并行编程的语言和编译器。它旨在提供一个基于 Python 的编程环境，以高效编写自定义 DNN 计算内核，并能够在现代 GPU 硬件上以最大吞吐量运行。  
简单讲，就是可以用Python写GPU算子。  
<!-- more -->


## 简单的Add算子

```python
import torch
import triton
import triton.language as tl

# 定义 kernel 函数，执行逐元素加法
@triton.jit
def add_kernel(
    x_ptr, y_ptr, output_ptr, n_elements,
    BLOCK_SIZE: tl.constexpr
):
    # 获取当前线程的 ID
    pid = tl.program_id(0)
    # 计算当前 block 内的索引
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    # 通过 masks 防止越界访问
    mask = offsets < n_elements
    # 从全局内存中加载 x 和 y 的元素
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)
    # 执行逐元素加法
    output = x + y
    # 将结果写回到全局内存
    tl.store(output_ptr + offsets, output, mask=mask)

data = [i for i in range(1, 1023)] # 特意设置了1022个元素，让128不整除

# 测试数据
x = torch.tensor(data, device='cuda')
y = torch.tensor(data, device='cuda')
output = torch.empty_like(x)

# 每个 block 的线程数
block_size = 128

grid = triton.cdiv(x.numel(), block_size)
# 启动 kernel，执行逐元素加法
add_kernel[(grid,)](x, y, output, x.numel(), BLOCK_SIZE=block_size)

print(output)

# 检查结果是否正确
print(torch.allclose(output, x + y))
```
运行上面代码，可以得到结果：
```
tensor([   2,    4,    6,  ..., 2040, 2042, 2044], device='cuda:0')
True
```

### 关键部分解释
* @triton.jit 装饰器：这个装饰器将 Python 函数 JIT 编译为 Triton kernel。函数的输入可以是张量指针、整数或编译时常量。
* tl.program_id(0)：返回当前线程块的唯一 ID，类似于 CUDA 中的 blockIdx.x。
* tl.arange(0, BLOCK_SIZE)：为当前线程块中的每个线程分配索引。
* x.numel()表示x的元素个数，`triton.cdiv(x.numel(), block_size)`就是向上取整，保证grid数量足够。
* tl.load() 和 tl.store()：用于从全局内存加载数据和将数据存储回全局内存。
* mask：用来防止越界访问（即确保线程访问的数据在张量范围内）。


## Softmax算子
### Naive Softmax实现
首先，使用pytorch实现一个row-wise的naive softmax:
```python
import torch

def naive_softmax(x):
    """Compute row-wise softmax of X using native pytorch

    We subtract the maximum element in order to avoid overflows. Softmax is invariant to
    this shift.
    """
    # read  MN elements ; write M  elements; 读取MN元素；写M个元素
    x_max = x.max(dim=1)[0]
    # read MN + M elements ; write MN elements; 读取MN+M元素；写入MN元素
    z = x - x_max[:, None]
    # read  MN elements ; write MN elements; 读取MN元素；写入MN元素
    numerator = torch.exp(z)
    # read  MN elements ; write M  elements; 读取MN元素；写M个元素
    denominator = numerator.sum(dim=1)
    # read MN + M elements ; write MN elements; 读取MN M元素；写入MN元素
    ret = numerator / denominator[:, None]
    # in total: read 5MN + 2M elements ; wrote 3MN + 2M elements;
    return ret # 共：读取5MN+2M元素；写了3MN+2M个元素
```
为什么叫 “naive”（朴素）？
* 它使用了 多个中间张量（x_max, z, numerator, denominator, ret），每个都占用显存。
* 每一步都是独立的 PyTorch 操作，无法融合（kernel fusion），导致多次读写全局内存。
* 在 GPU 上，这会带来 较高的内存带宽压力 和 较低的计算效率。

### N较小的Triton版softmax
```python
import triton
import triton.language as tl
import torch

@triton.jit
def softmax_kernel(
    output_ptr,        # 输出指针
    input_ptr,         # 输入指针
    input_row_stride,  # 输入每行的步长（以元素为单位）
    output_row_stride, # 输出每行的步长
    n_cols,            # 每行的列数 N
    BLOCK_SIZE: tl.constexpr,  # 每个线程块处理的列数
):
    # 获取当前处理的行索引
    row_idx = tl.program_id(0)
    
    # 计算该行在全局内存中的起始地址
    row_start_ptr = input_ptr + row_idx * input_row_stride
    
    # 使用向量化加载（如果 BLOCK_SIZE 是 128 的倍数等）
    col_offsets = tl.arange(0, BLOCK_SIZE)
    input_ptrs = row_start_ptr + col_offsets
    
    # 掩码：防止越界访问
    mask = col_offsets < n_cols
    
    # 加载输入行
    row = tl.load(input_ptrs, mask=mask, other=-float('inf'))
    
    # 数值稳定：减去最大值
    row_max = tl.max(row, axis=0)
    row_minus_max = row - row_max
    
    # 计算 exp
    exp_row = tl.exp(row_minus_max)
    
    # 求和
    exp_sum = tl.sum(exp_row, axis=0)
    
    # 归一化
    softmax_output = exp_row / exp_sum
    
    # 存储结果
    output_row_start = output_ptr + row_idx * output_row_stride
    output_ptrs = output_row_start + col_offsets
    tl.store(output_ptrs, softmax_output, mask=mask)

def softmax(x: torch.Tensor):
    """
    对输入张量 x 的最后一维执行 softmax。
    假设 x 是 2D 张量 (M, N)
    """
    assert x.is_cuda, "Input must be on GPU"
    assert x.dim() == 2, "Only 2D tensors supported for now"
    
    M, N = x.shape
    
    # 选择合适的 BLOCK_SIZE（必须是 2 的幂，且 <= 4096）
    BLOCK_SIZE = triton.next_power_of_2(N)
    BLOCK_SIZE = min(BLOCK_SIZE, 4096)  # Triton 限制最大 BLOCK_SIZE
    
    # 分配输出张量
    output = torch.empty_like(x)
    
    # 启动内核
    grid = (M,)  # 每行一个程序
    softmax_kernel[grid](
        output,
        x,
        x.stride(0),   # input_row_stride
        output.stride(0),  # output_row_stride
        N,
        BLOCK_SIZE=BLOCK_SIZE,
    )
    
    return output
```

### 说明
* 数值稳定性：通过减去每行最大值避免 exp 溢出。
* 并行性：每行由一个 Triton 程序（program）处理，利用 BLOCK_SIZE 个线程并行加载/计算该行。
* 内存访问：使用 mask 避免越界读写，适用于 N 不是 BLOCK_SIZE 整数倍的情况。
* BLOCK_SIZE：自动选择最接近 N 的 2 的幂，但不超过 4096（Triton 的限制）。
