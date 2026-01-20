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

---

## 简单模块调用例子
使用 **Icarus Verilog（iverilog）** 时，目录结构可以非常简单，因为它是命令行工具，不强制要求特定项目结构。但为了清晰和可维护性，建议采用如下组织方式：

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
