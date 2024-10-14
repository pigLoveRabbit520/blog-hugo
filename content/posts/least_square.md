---
title: 最小二乘法和线性回归
author: pigLoveRabbit
tags: []
categories:
  - Machine Learning
date: 2024-10-08 14:00:00
---
## 最小二乘法
早在19世纪,勒让德就认为让"误差的平方和最小"估计出来的模型是最接近真实情形的。  
按照勒让德的最佳原则,于是就是求:  
$$
\text{L} = \sum_{i=1}^{n} \left( y_i - f(x_i) \right)^2
$$
这个目标函数取得最小值时的函数参数,这就是最小二乘法的思想想,所谓"二乘"就是平方的意思。从这里我们可以看到,**最小二乘法其实
就是用来做函数拟合的一种思想**。  
至于怎么求出具体的参数那就是另外一个问题了,理论上可以用导数法、几何法,工程上可以用**梯度下降法**。下面以最常用的线性回归为
例进行推导和理解。  
在**机器学习**中用于回归问题的损失函数(Loss Function)是均方误差(MSE)：
$$
\text{L} = \frac{1}{2n} \sum_{i=1}^{n} \left( y_i - f(x_i) \right)^2
$$
其实就是多了个1/2n。

## 线性回归
线性回归因为比较简单,可以直接推导出解析解,而且许多非线性的问题也可以转化为线性问题来解决,所以得到了广泛的应用。甚至许多人认为最小二乘法指的就是线性回归,其实并不是,最小二乘法就是一种思想,它可以拟合任意函数,线性回归只是其中一个比较简单而且也很常用的函数,所以讲最小二乘法基本都会以它为例。  
下面我会先用矩阵法进行推导,然后再用几何法来帮助你理解最小二乘法的几何意义。


设计矩阵 **X** （维度为$ n \times \left(p + 1\right) $）：包含所有样本的特征信息，第一列是全为 1 的常数列，代表截距项$ \beta_{0} $ ，其余列为各个特征的值。
$$
\boldsymbol{X}=\left[\begin{array}{ccccc}
1 & x_{11} & x_{12} & \ldots & x_{1 p} \newline
1 & x_{21} & x_{22} & \ldots & x_{2 p} \newline
\vdots & \vdots & \vdots & \ddots & \vdots \newline
1 & x_{n 1} & x_{n 2} & \ldots & x_{n p}
\end{array}\right]
$$

回归系数向量 $ \beta $ （维度为$ \left(p + 1\right) \times 1 $）：包含所有的回归系数（包括截距项）。  
$$
\boldsymbol{\beta} = \begin{bmatrix}
\beta_0 \newline
\beta_1 \newline
\beta_2 \newline
\vdots \newline
\beta_p
\end{bmatrix}
$$
观测值向量$ y $（维度为$ n \times 1 $）：包含所有样本的目标值。  
$$
\boldsymbol{y} = \begin{bmatrix}
y_0 \newline
y_1 \newline
y_2 \newline
\vdots \newline
y_p
\end{bmatrix}
$$
线性回归的模型可以简化为：
$$
\boldsymbol{y} = \boldsymbol{X}\boldsymbol{\beta} + \boldsymbol{\epsilon}
$$
其中$\boldsymbol{\epsilon}$是误差向量。  
为了估计$\beta$，我们通常使用 **最小二乘法** 来最小化残差平方和，即：
$$
\min_{\beta} || \boldsymbol{y} - \boldsymbol{X}\boldsymbol{\beta} ||^2
$$
这里的$\|\cdot\|$表示Frobenius 范数或F-范数，是一种矩阵范数。  
矩阵A的Frobenius范数定义为矩阵A各项元素的绝对值平方的总和，即 ：
$$
||A||_F = \sqrt{\sum_{i,j} |a_{ij}|^2}
$$
这里$ a_{ij} $是矩阵 中第i行，第j列的元素。下标2可以省略，所以可以直接写成$||A||$。  
$||A||^2$其实是一个数，即行向量乘以列向量，我们可以这样改写上式：
$$
||A||^2 = A^TA
$$
所以误差函数可以表示为:
$$
|| y - X\beta ||^2 = (y - X\beta)^T (y - X\beta)
$$
展开后得到：
$$
(y - X\beta)^T (y - X\beta) = y^T y - y^T X\beta - (X\beta)^T y + (X\beta)^T X\beta
$$
然后得到：
$$
(y - X\beta)^T (y - X\beta) = y^T y - y^T X\beta - \beta^TX^T y + \beta^T X^T X \beta
$$
这里$y^T X\beta $和$ \beta^TX^T y$ 都是$1 \times 1$的标量，对于标量，有$a^T = a$，因此$\beta^TX^T y = (\beta^TX^T y)^T = y^TX\beta $  
合并同类项：
$$
(y - X\beta)^T (y - X\beta) = y^T y - 2\beta^T X^Ty + \beta^T X^T X \beta
$$




参考：
* [矩阵范数](https://sunocean.life/blog/blog/2020/08/31/deep-learning-math-norm)
* [最小二乘法线性回归：矩阵视角](https://zhuanlan.zhihu.com/p/33899560)