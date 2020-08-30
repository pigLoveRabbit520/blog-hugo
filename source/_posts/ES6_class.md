title: ES6中的class
author: Salamander
tags:
  - JavaScript
  - class
categories:
  - JavaScript
date: 2020-08-29 12:00:00
---
![](https://s1.ax1x.com/2020/08/30/dbt2p6.png)

## ES6
ECMAScript 2015或ES2015是对JavaScript编程语言的重大更新。这是自2009年对ES5进行标准化以来对语言的首次重大更新，ES6加入很多有用的特性。因此，ES2015通常被称为ES6。

本文环境：
* NodeJs版本：v12.13.0
* OS：Ubuntu 18.04.4 LTS

<!-- more -->


## class语法糖


首先我们先来看一下关于 ES6 中的类
```JavaScript
class Persion {
    say() {
        console.log('hello~')
    }
}
```
上面这段代码是 ES6 中定义一个类的写法，其实只是一个语法糖，而实际上当我们给一个类添加一个属性的时候，会调用到 `Object.defineProperty` 这个方法，它会接受三个参数：target 、name 和 descriptor ，所以上面的代码实际上在执行时是这样的：
```JavaScript
function Persion() {}

Object.defineProperty(Persion.prototype, "say", {
    value: function() { console.log("hello ~"); },
    enumerable: false,
    configurable: true,
    writable: true
});
```

## ES6中extends
ES6中可以很方便地用`extends`的实现继承：
```JavaScript
class Shape {
    constructor(x, y) {
        this.x = x;
        this.y = y;
    }

    move(x, y) {
        this.x += x;
        this.y += y;
        console.info('Shape moved.');
    }
}

class Rectangle extends Shape {
    constructor() {
        super();  // ES6 要求，子类的构造函数必须执行一次 super 函数，否则会报错。
    }
}
```
要用ES5实现继承，就是用利用**原型链**。
```JavaScript
// Shape - 父类(superclass)
function Shape() {
  this.x = 0;
  this.y = 0;
}

// 父类的方法
Shape.prototype.move = function(x, y) {
  this.x += x;
  this.y += y;
  console.info('Shape moved.');
};

// Rectangle - 子类(subclass)
function Rectangle() {
  Shape.call(this); // call super constructor.
}

// 子类续承父类
Rectangle.prototype = Object.create(Shape.prototype);
Rectangle.prototype.constructor = Rectangle;

var rect = new Rectangle();

console.log('Is rect an instance of Rectangle?',
  rect instanceof Rectangle); // true
console.log('Is rect an instance of Shape?',
  rect instanceof Shape); // true
rect.move(1, 1); // Outputs, 'Shape moved.'
```









参考：
* [MDN Object create](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/create)