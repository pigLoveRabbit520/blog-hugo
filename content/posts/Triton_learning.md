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

# 测试
M, N = 128, 512
x = torch.randn(M, N, device='cuda')

y_torch = torch.softmax(x, dim=-1)
y_triton = softmax(x)

# 检查数值误差
print("Max diff:", (y_torch - y_triton).abs().max().item())
assert torch.allclose(y_torch, y_triton, atol=1e-5), "Results don't match!"
print("✅ Triton softmax matches PyTorch!")
```

### 说明
* 数值稳定性：通过减去每行最大值避免 exp 溢出。
* 并行性：每行由一个 Triton 程序（program）处理，利用 BLOCK_SIZE 个线程并行加载/计算该行。
* 内存访问：使用 mask 避免越界读写，适用于 N 不是 BLOCK_SIZE 整数倍的情况。
* BLOCK_SIZE：自动选择最接近 N 的 2 的幂，但不超过 4096（Triton 的限制）。

## 为啥需要input_row_stride这样的参数
`input_row_stride`（以及 `output_row_stride`）参数的存在，是为了让 Triton 内核能够**正确处理非连续内存布局（non-contiguous tensors）**，尤其是**跨步（strided）张量**。

---

### 1. 什么是 stride（步长）？

在 PyTorch（以及大多数张量库）中，一个张量在内存中是**一维线性存储**的，但通过 `shape` 和 `stride` 来解释其多维结构。

例如：
```python
x = torch.randn(4, 8)  # shape=(4,8)
print(x.stride())      # 通常是 (8, 1)
```
- `stride[0] = 8`：表示第 0 维（行）每增加 1，内存地址跳过 8 个元素。
- `stride[1] = 1`：表示第 1 维（列）每增加 1，内存地址跳过 1 个元素。

所以，`x[i, j]` 对应的内存偏移是：`i * stride[0] + j * stride[1]`。

---

### 2. 为什么不能假设 stride 是固定的？

因为 PyTorch 张量**不一定是连续的**！例如：

#### 情况一：转置（transpose）
```python
x = torch.randn(4, 8)
y = x.t()  # shape=(8,4), stride=(1, 8) ← 注意！
```
- 现在 `y` 的第 0 维（原列）的 stride 是 1，第 1 维（原行）的 stride 是 8。
- 如果你的内核假设“每行连续存储（stride=列数）”，那么对 `y` 调用 softmax 就会出错！

#### 情况二：切片（slice）
```python
x = torch.randn(10, 20)
z = x[::2, :]  # 取偶数行，shape=(5,20)，但 stride[0] = 40（不是20！）
```

#### 情况三：view / reshape 后的非连续张量
某些 reshape 操作可能产生非连续内存布局。

---

### 3. 如果不传 stride 会发生什么？

假设你写死：
```python
row_start_ptr = input_ptr + row_idx * n_cols  # ❌ 错误！
```

这**只在张量是行连续（row-major, contiguous）时才正确**。一旦遇到转置或切片张量，就会：
- 读取错误的内存位置
- 导致结果完全错误
- 甚至越界访问（segmentation fault）

---

### 4. 正确做法：使用实际 stride

通过传入 `x.stride(0)`（即第 0 维的 stride），Triton 内核可以正确计算每一行的起始地址：

```python
row_start_ptr = input_ptr + row_idx * input_row_stride
```

这样无论张量是否连续、是否转置、是否切片，都能正确访问数据。

---

### 5. 实际例子对比

```python
x = torch.randn(2, 4, device='cuda')
y = x.t()  # shape=(4,2), stride=(1, 4)

# 正确调用（传 stride）：
softmax(y)  # 内核使用 y.stride(0) == 1，正确！

# 错误实现（假设 stride = n_cols = 2）：
# 会认为第0行在 0*2=0，第1行在 1*2=2，...
# 但实际上第1行在内存偏移 1*1 = 1！
```

---

### 6. 补充：为什么 output 也需要 stride？

因为输出张量 `output = torch.empty_like(x)` 会**继承输入的内存布局**（包括 stride）。所以你也必须用 `output.stride(0)` 来正确写入结果。

---
但其实我这个Softmax算子，更通用一点，也应该加入列的stride。


## mean算子
这个例子有点复杂
```
import torch
import triton
import triton.language as tl


@triton.jit
def mean_kernel(
    input_ptr,
    output_ptr,
    input_stride0,
    input_stride1,
    input_stride2,
    output_stride0,
    output_stride1,
    M,  # size before reduction dim
    N,  # size of reduction dim
    K,  # size after reduction dim
    BLOCK_SIZE: tl.constexpr,
):
    """
    Kernel for computing mean along a single dimension.
    Input is viewed as (M, N, K) where N is the dimension being reduced.
    """
    # Program ID gives us which output element we're computing
    pid = tl.program_id(0)

    # Compute output indices
    m_idx = pid // K
    k_idx = pid % K

    # Bounds check
    if m_idx >= M or k_idx >= K:
        return

    # Accumulate sum across reduction dimension
    acc = 0.0
    for n_start in range(0, N, BLOCK_SIZE):
        n_offsets = n_start + tl.arange(0, BLOCK_SIZE)
        mask = n_offsets < N

        # Calculate input indices
        input_idx = (
            m_idx * input_stride0 + n_offsets * input_stride1 + k_idx * input_stride2
        )

        # Load and accumulate
        vals = tl.load(input_ptr + input_idx, mask=mask, other=0.0)
        acc += tl.sum(vals)

    # Compute mean and store
    mean_val = acc / N
    output_idx = m_idx * output_stride0 + k_idx * output_stride1
    tl.store(output_ptr + output_idx, mean_val)


def mean_dim(
    input: torch.Tensor,
    dim: int,
    keepdim: bool = False,
    dtype: torch.dtype | None = None,
) -> torch.Tensor:
    """
    Triton implementation of torch.mean with single dimension reduction.

    Args:
        input: Input tensor
        dim: Single dimension along which to compute mean
        keepdim: Whether to keep the reduced dimension
        dtype: Output dtype. If None, uses input dtype (or float32 for integer inputs)

    Returns:
        Tensor with mean values along specified dimension
    """
    # Validate inputs
    assert input.is_cuda, "Input must be a CUDA tensor"
    assert (
        -input.ndim <= dim < input.ndim
    ), f"Invalid dimension {dim} for tensor with {input.ndim} dimensions"

    # Handle negative dim
    if dim < 0:
        dim = dim + input.ndim

    # Handle dtype
    if dtype is None:
        if input.dtype in [torch.int8, torch.int16, torch.int32, torch.int64]:
            dtype = torch.float32
        else:
            dtype = input.dtype

    # Convert input to appropriate dtype if needed
    if input.dtype != dtype:
        input = input.to(dtype)

    # Get input shape and strides
    shape = list(input.shape)

    # Calculate dimensions for kernel
    M = 1
    for i in range(dim):
        M *= shape[i]

    N = shape[dim]

    K = 1
    for i in range(dim + 1, len(shape)):
        K *= shape[i]

    # Reshape input to 3D view (M, N, K)
    input_3d = input.reshape(M, N, K)

    # Create output shape
    if keepdim:
        output_shape = shape.copy()
        output_shape[dim] = 1
    else:
        output_shape = shape[:dim] + shape[dim + 1 :]

    # Create output tensor
    output = torch.empty(output_shape, dtype=dtype, device=input.device)

    # Reshape output for kernel
    if keepdim:
        output_2d = output.reshape(M, 1, K).squeeze(1)
    else:
        output_2d = output.reshape(M, K)

    # Launch kernel
    grid = (M * K,)
    BLOCK_SIZE = 1024

    mean_kernel[grid](
        input_3d,
        output_2d,
        input_3d.stride(0),
        input_3d.stride(1),
        input_3d.stride(2),
        output_2d.stride(0),
        output_2d.stride(1) if output_2d.ndim > 1 else 0,
        M,
        N,
        K,
        BLOCK_SIZE,
    )

    return output


def mean_batch_invariant(input, dim, keepdim=False, dtype: torch.dtype | None = None):
    assert dtype is None or dtype == torch.float32, f"unsupported dtype: {dtype}"
    if len(dim) == 1:
        return mean_dim(input, dim[0], keepdim=keepdim)
    else:
        assert input.dtype in {
            torch.float16,
            torch.bfloat16,
            torch.float32,
        }, "only float types supported for now"
        n_elems = 1
        for d in dim:
            n_elems *= input.shape[d]
        return torch.sum(input, dim=dim, keepdim=keepdim, dtype=torch.float32) / n_elems


def test_mean_batch_invariant():
    # 设置随机种子以确保结果可重复
    torch.manual_seed(42)

    # 定义测试的维度和大小
    test_cases = [
        # 一维张量
        (1024, ),
        # # 二维张量
        # (512, 512),
        # # 三维张量
        # (16, 32, 2048),
        # (2048, 4096, 64),
    ]

    # 定义允许的误差范围
    atol = 1e-3
    rtol = 1e-3

    for shape in test_cases:
        # 生成随机测试数据
        input_tensor = torch.randn(*shape, dtype=torch.float32, device='npu:0')
        input_dim = input_tensor.ndim
        # print(f"input = {input_tensor}, dim = {input_dim}")

        # 测试指定维度均值
        for dim in range(input_dim):
            # 精度测试
            output_torch = torch.mean(input_tensor, dim=dim, keepdim=True)
            output_triton = mean_batch_invariant(input_tensor, dim=[dim], keepdim=True)
            assert torch.allclose(output_torch, output_triton, atol=atol, rtol=rtol), \
                f"Test failed for shape {shape}. The difference between torch and triton on dim {dim} is {torch.abs(output_torch - output_triton)}."

            # 性能测试
            # t_torch = triton.testing.do_bench(lambda: torch.mean(input_tensor, dim=dim, keepdim=True))
            # t_triton = triton.testing.do_bench(lambda: mean_batch_invariant(input_tensor, dim=[dim], keepdim=True))
            # print(f"Dim {dim}: runtime of torch = {t_torch} ms, runtime of triton = {t_triton} ms, times = {t_triton / t_torch}")

        print(f"Test passed for shape {shape}.")


if __name__ == "__main__":
    test_mean_batch_invariant()

```

在 PyTorch 中，`torch.reshape()` 的行为类似于 NumPy 的 `reshape`：**它尽可能返回一个 view（视图）**，但在某些情况下**无法返回 view 时，会返回一个 copy（副本）**。

### 什么时候会返回 copy？

当张量的内存布局（由 `stride` 决定）不允许在不复制数据的情况下形成目标形状时，`reshape` 就必须返回一个 copy。

具体来说，如果原张量不是**连续的（contiguous）**，并且目标形状无法通过调整 stride 来表示，那么 `reshape` 就会触发数据复制。

---

### 举个例子

```python
import torch

# 创建一个非连续的张量
x = torch.arange(12).reshape(3, 4)   # shape: (3, 4)
y = x.t()                            # 转置 -> shape: (4, 3)，但内存不连续

print("y.is_contiguous():", y.is_contiguous())  # False

# 尝试 reshape
z = y.reshape(12)

print("z.is_contiguous():", z.is_contiguous())  # True

# 检查是否是 view：修改 z 是否会影响 y？
# 注意：如果 z 是 y 的 view，那么修改 z 会影响 y（或底层数据）
# 但 reshape 返回的 z 是 copy，所以不会影响 y

z[0] = -999
print("y[0, 0]:", y[0, 0])  # 仍然是 0，说明 z 是 copy
```

输出类似：
```
y.is_contiguous(): False
z.is_contiguous(): True
y[0, 0]: 0
```

这说明：因为 `y` 是非连续的（由 `.t()` 产生），`y.reshape(12)` 无法只通过修改 stride 来展平，所以 PyTorch **必须复制数据**，返回一个新张量（copy）。

---

### 对比：连续张量的 reshape（返回 view）

```python
a = torch.arange(12)          # contiguous
b = a.reshape(3, 4)           # view
b[0, 0] = 888
print(a[0])  # 输出 888，说明是 view
```

---

### 总结

- `torch.reshape()` **优先返回 view**；
- **当输入张量不连续且目标形状无法通过 stride 表达时，会返回 copy**；
- 判断是否是 view 的一个实用方式：检查 `.is_contiguous()` 或通过修改元素看是否影响原张量（但后者不推荐用于生产代码）；
- 如果你**明确需要 view**，可使用 `.view()`：它在无法创建 view 时会报错；
- 如果你**明确需要新内存**，可以用 `.reshape().clone()` 或先 `.contiguous()` 再 `.view()`。
