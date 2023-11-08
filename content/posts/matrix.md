---
title: 几何中矩阵
author: pigLoveRabbit
tags: []
categories:
  - Matrix
  - css
date: 2022-07-10 14:00:00
---
## css中应用
对元素的`transform`属性，我们可以应用矩阵
```
matrix(a,b,c,d,e,f)	
```
这6参数，对应的矩阵就是：  

![upload successful](/images/matrix1.png)  
注意书写方向是**竖着的**。  e, f参数其实就是x，y方向上偏移。

<!-- more -->

我们知道平面中旋转的矩阵是

![upload successful](/images/matrix_rotate.png)  
那我们可以写个demo了
```
<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
.square {
  height: 100px;
  width: 100px;
  background-color: red;
  transform: matrix(1, 0, 0, 1, 30, 30);
}
</style>
</head>
<body>

<h2>Square CSS</h2>
<div class="square" id="square"></div>
</body>
<script>
    let angle = 0; // 弧度
    let squareElement = document.getElementById("square");
    function getMatrix(angle) {
        return [Math.cos(angle), Math.sin(angle), -Math.sin(angle), Math.cos(angle), 0, 0];
    }
    setInterval(() => {
        angle++;
        let items = getMatrix(angle).join(",");
        squareElement.style.transform = `matrix(${items})`;
    }, 80);
</script>
</html> 
```
上面展示了一个自动利用矩阵旋转的方块。



参考：
* [理解CSS3 transform中的Matrix(矩阵)](https://www.zhangxinxu.com/wordpress/2012/06/css3-transform-matrix-%E7%9F%A9%E9%98%B5/)
* [如何通俗地讲解「仿射变换」这个概念？](https://www.zhihu.com/question/20666664/answer/157400568)