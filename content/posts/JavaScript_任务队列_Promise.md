---
title: JavaScript中的微任务、宏任务和Promise
author: pigLoveRabbit
tags:
  - Promise
  - Nodejs
categories:
  - Nodejs
date: 2024-09-18 09:00:00
---
JavaScript会将异步任务划分为微任务和宏任务，微任务会在宏任务之前执行（因为每次从主线程切换到任务队列时，都会优先遍历微任务队列，后遍历宏任务队列）。

<!-- more -->
## 一个小例子
```
// 1
setTimeout(() => { // 宏任务
    console.log(4)
}, 0)

// 2
new Promise(resolve => {
    resolve()
    console.log(1)
    // 3
}).then(data => {  // 微任务
    console.log(3)
})

// 4
console.log(2) // 同步任务

// 执行结果： 1 2 3 4
```
