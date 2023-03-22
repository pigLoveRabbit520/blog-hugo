title: OpenGL中旋转矩阵
mathjax: true
author: pigLoveRabbit
tags:
  - 矩阵
categories:
  - 矩阵
date: 2023-03-21 11:00:00
---
## 旋转矩阵
首先要说明的是，左右手旋转矩阵是不一样的，DirectX中是左手坐标系，而OpenGL用的是**右手坐标系**，这里给出的旋转矩阵也是基于右手坐标系的  

绕x轴旋转矩阵：  


$$
\begin{bmatrix} 
\color{red}1 & \color{red}0 & \color{red}0 & \\ 
\color{green}0 & \color{green}{\cos \theta} & - \color{green}{\sin \theta} \\  
\color{blue}0 & \color{blue}{\sin \theta} & \color{blue}{\cos \theta} \\  \end{bmatrix}
$$


沿y轴旋转矩阵：  
$$
\begin{bmatrix} 
\color{red}{\cos \theta} & \color{red}0 & \color{red}{\sin \theta} \\ 
\color{green}0 & \color{green}1 & \color{green}0  \\ 
-\color{blue}{\sin \theta} & \color{blue}0 & \color{blue}{\cos \theta}
\end{bmatrix}
$$


沿z轴旋转矩阵：  
$$
\begin{bmatrix} 
\color{red}{\cos \theta} & - \color{red}{\sin \theta} & \color{red}0  \\ 
\color{green}{\sin \theta} & \color{green}{\cos \theta} & \color{green}0  \\ 
\color{blue}0 & \color{blue}0 & \color{blue}1  \\ 
\end{bmatrix}
$$










参考：  
* [变换](https://learnopengl-cn.readthedocs.io/zh/latest/01%20Getting%20started/07%20Transformations/#_18)