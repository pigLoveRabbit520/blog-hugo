---
title: Attention机制学习
author: pigLoveRabbit
tags:
  - Blender
categories:
  - Blender
date: 2024-12-01 20:00:00
---
![logo](https://ask.qcloudimg.com/http-save/yehe-1599485/a7ad1d33a72390f3e6f52a85701f0329.png)

<!-- more -->

## Q K V
在深度学习中，很多 LLM 的训练都使用 Transformer 架构，而在 Transformer 架构中计算的过程涉及到的最关键的就是注意力，它是整个过程中重要的基础。注意力抽象出了 3 个重要的概念，在计算过程中对应着 3 个矩阵，如下所示：

* Query：在自主提示下，自主提示的内容，对应着矩阵 Q
* Keys：在非自主提示下，进入视觉系统的线索，对应着矩阵 K
* Values：使用 Query 从 Keys 中匹配得到的线索，基于这些线索得到的进入视觉系统中焦点内容，对应着矩阵 V

























[参考](http://shiyanjun.cn/archives/2688.html)
