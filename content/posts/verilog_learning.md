---
title: Verilog学习
author: pigLoveRabbit
tags:
  - pytorch
categories:
  - verilog
  - 硬件
date: 2026-01-18 21:00:00
---
![pytorch](/images/verilog_eg.png)
<!-- more -->

## 基本语法
Verilog HDL（简称 Verilog ）是一种硬件描述语言，用于数字电路的系统设计。可对算法级、门级、开关级等多种抽象设计层次进行建模。  
Verilog 继承了 C 语言的多种操作符和结构，与另一种硬件描述语言 VHDL 相比，语法不是很严格，代码更加简洁，更容易上手。  
Verilog 不仅定义了语法，还对语法结构都定义了清晰的仿真语义。因此，Verilog 编写的数字模型就能够使用 Verilog 仿真器进行验证。
当然可以！以下是 **Verilog HDL（硬件描述语言）的基础语法总结**，适合初学者快速掌握核心概念，也方便你后续进行数字电路设计、仿真和综合。

---

## 🧱 一、基本结构

### 1. 模块（Module）—— 设计的基本单元
```verilog
module module_name (
    input  wire [N-1:0] in1,
    input  wire clk,
    output reg  [M-1:0] out
);
    // 内部逻辑：assign、always、实例化等
endmodule
```
- 所有设计都从 `module ... endmodule` 开始。
- 端口列表可选（空模块可不写括号内容）。
- 端口默认类型：`input` → `wire`，`output` → `wire`（除非显式声明为 `reg`）。

---

## 🔤 二、数据类型

### 1. `wire`（线网型）
- 表示**结构连接**，不能存储值。
- 由 `assign`、门级元件或模块输出驱动。
- 默认 1 位，多位需 `[N-1:0]` 声明。
```verilog
wire a;
wire [7:0] data;
assign a = b & c;
```

### 2. `reg`（寄存器型）
- **不是**一定对应硬件寄存器！
- 只能在 `always` 或 `initial` 块中赋值。
- 用于过程赋值（时序或组合逻辑）。
```verilog
reg [3:0] counter;
always @(posedge clk) counter <= counter + 1;
```

> ✅ 组合逻辑用 `reg`？可以！只要在 `always @(*)` 中，综合工具会生成组合逻辑。

---

## ⚙️ 三、赋值方式

### 1. 连续赋值（Continuous Assignment）
```verilog
assign y = a & b;  // 仅用于 wire
```

### 2. 过程赋值（Procedural Assignment）
- 在 `always` 或 `initial` 中使用：
  - **阻塞赋值 `=`**：顺序执行（常用于组合逻辑）
  - **非阻塞赋值 `<=`**：并行更新（**必须用于时序逻辑**）

```verilog
always @(*) begin
    y = a + b;      // 阻塞，组合逻辑
end

always @(posedge clk) begin
    q <= d;         // 非阻塞，时序逻辑（触发器）
end
```

---

## 🔄 四、常用语句块

### 1. `always` 块
- 描述**时序或组合逻辑**。
- 敏感列表：
  - `always @(posedge clk)` → 时序
  - `always @(posedge clk or negedge rst_n)` → 带异步复位
  - `always @(*)` → 自动包含所有输入（推荐用于组合逻辑）

### 2. `initial` 块
- 仅用于**仿真**（testbench），不可综合。
- 通常用于初始化、产生激励。

```verilog
initial begin
    clk = 0;
    forever #5 clk = ~clk;  // 10ns 周期时钟
end
```

---

## 🔢 五、数值表示

| 写法          | 含义                     |
|---------------|--------------------------|
| `1'b0`        | 1 位二进制 0             |
| `4'b1010`     | 4 位二进制 1010          |
| `8'hFF`       | 8 位十六进制 FF（=255）  |
| `16'd100`     | 16 位十进制 100          |
| `32'o77`      | 32 位八进制 77           |
| `'b101`       | 默认宽度（通常 32 位）   |

> 默认宽度由工具决定，**建议显式指定位宽**！

---

## 🧮 六、运算符

| 类型         | 运算符                     | 说明                     |
|--------------|----------------------------|--------------------------|
| **按位**     | `&`, `|`, `^`, `~`         | 作用于每一位             |
| **归约**     | `&a`, `|a`, `^a`           | 将向量压缩为 1 位        |
| **逻辑**     | `&&`, `||`, `!`            | 返回 1-bit 布尔结果      |
| **关系**     | `==`, `!=`, `>`, `<`       |                          |
| **移位**     | `<<`, `>>`                 | 逻辑移位                 |
| **拼接**     | `{a, b, 4'b0}`             | 拼接操作符               |
| **条件**     | `c ? a : b`                | 三目运算符               |

---

## 🔌 七、模块实例化（调用）

### 1. 位置关联（不推荐）
```verilog
and_gate u1 (in1, in2, out);
```

### 2. **命名关联（推荐）**
```verilog
and_gate u1 (
    .a(in1),
    .b(in2),
    .y(out)
);
```

---

## 📦 八、参数化设计（可配置）

```verilog
module decoder #(
    parameter N = 3
)(
    input  wire [N-1:0] addr,
    output wire [(1<<N)-1:0] y
);
    // ...
endmodule

// 实例化时重定义参数
decoder #(.N(4)) u_dec (.addr(addr4), .y(y16));
```

---

## 🧪 九、Testbench 基础结构

```verilog
module tb_xxx;
    reg clk, rst;
    reg [7:0] data_in;
    wire [7:0] data_out;

    // DUT 实例化
    my_module u_dut (.clk(clk), .data_in(data_in), .data_out(data_out));

    // 时钟生成
    initial begin
        clk = 0;
        forever #5 clk = ~clk;
    end

    // 激励
    initial begin
        $dumpfile("wave.vcd");
        $dumpvars(0, tb_xxx);
        rst = 1; data_in = 0;
        #20 rst = 0;
        #10 data_in = 8'hAA;
        #100 $finish;
    end
endmodule
```

---

## ✅ 十、可综合 vs 不可综合

| 可综合（能变成硬件）       | 不可综合（仅仿真）         |
|----------------------------|----------------------------|
| `assign`                   | `$display`, `$monitor`     |
| `always @(posedge clk)`    | `$finish`, `$stop`         |
| 有限状态机、计数器         | `forever`, `#delay`        |
| 参数化模块                 | `initial`（除 RAM 初始化）|

> 💡 仿真可用任何语法，但**综合只支持子集**！

---

## 🎯 总结口诀

- **`wire` 连线，`reg` 赋值**  
- **组合 `@(*)`，时序 `@(posedge clk)`**  
- **时序用 `<=`，组合用 `=`**  
- **模块用命名关联，参数用 `#()`**  
- **testbench 用 `initial` + `$dumpfile`**
---

## 简单模块调用例子
这里使用 **Icarus Verilog（iverilog）** 做仿真工具，[下载地址](https://bleyer.org/icarus/)  
目录结构可以非常简单，因为它是命令行工具，不强制要求特定项目结构。但为了清晰和可维护性，建议采用如下组织方式：

---

### 📁 推荐的目录结构（针对你的 01 示例）

```bash
your_project/
├── src/               # 存放设计源文件（Design）
│   ├── and_gate.v
│   └── top.v
└── tb/                # 存放测试平台（Testbench）
    └── tb_top.v
```

或者更简单的扁平结构（适合小项目）：

```bash
your_project/
├── and_gate.v
├── top.v
└── tb_top.v
```

> ✅ 对于你这个小例子，**扁平结构完全足够**。

---

### 🔧 具体操作步骤（以扁平结构为例）

#### 1. 创建三个文件

- `and_gate.v`
- `top.v`
- `tb_top.v`

内容分别如下：

##### ▶ `and_gate.v`
```verilog
module and_gate (
    input  wire a,
    input  wire b,
    output wire y
);
    assign y = a & b;
endmodule
```

##### ▶ `top.v`
```verilog
module top (
    input  wire in1,
    input  wire in2,
    output wire out
);
    and_gate u_and (
        .a(in1),
        .b(in2),
        .y(out)
    );
endmodule
```

##### ▶ `tb_top.v`
```verilog
`timescale 1ns / 1ps

module tb_top;
    reg  in1;
    reg  in2;
    wire out;

    top u_dut (
        .in1(in1),
        .in2(in2),
        .out(out)
    );

    initial begin
        $display("Time\tin1\tin2\tout");
        $monitor("%0t\t%b\t%b\t%b", $time, in1, in2, out);
        in1 = 0; in2 = 0; #10;
        in1 = 0; in2 = 1; #10;
        in1 = 1; in2 = 0; #10;
        in1 = 1; in2 = 1; #10;
        $finish;
    end
endmodule
```

---

#### 2. 打开终端，进入该目录

```bash
cd your_project
```

#### 3. 编译所有 Verilog 文件

```bash
iverilog -o sim top.v and_gate.v tb_top.v
```

> 💡 顺序无关紧要，但 **testbench 必须包含顶层仿真模块（这里是 `tb_top`）**，而 `iverilog` 会自动解析模块依赖。

你也可以简写为：
```bash
iverilog -o sim *.v
```

#### 4. 运行仿真

```bash
vvp sim
```

#### 5. 查看输出（预期）

```
Time    in1     in2     out
0       0       0       0
10      0       1       0
20      1       0       0
30      1       1       1
```

✅ 成功！



参考:  
* [verilog-project-1](https://digilent.com/reference/learn/software/tutorials/verilog-project-1/start)
