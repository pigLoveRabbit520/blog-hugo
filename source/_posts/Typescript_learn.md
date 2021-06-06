title: Typescript学习
author: Salamander
tags:
  - Typescript
categories:
  - Javascript
date: 2021-06-05 16:00:00
---
## 安装
安装TypeScript还是很简单的：
```
npm install -g typescript
```
写个hello.ts
```
function sayHello(person: string) {
    return 'Hello, ' + person;
}

let user = 'Tom';
console.log(sayHello(user));
```
然后执行
```
tsc hello.ts
```
这时候会生成一个编译好的文件 `hello.js`：
```
function sayHello(person) {
    return 'Hello, ' + person;
}
var user = 'Tom';
console.log(sayHello(user));
```
可以看到，编译好后的就是平常的js代码，TS的一个优点在于类型的声明，这样在编译期bug就能及早发现（当然还有其他好处，例如**泛型**，**Enum**）。  
注意： **TypeScript** 只会在编译时对类型进行静态检查，如果发现有错误，编译的时候就会报错。而在运行时，与普通的 JavaScript 文件一样，不会对类型进行检查。



































参考：
* [TypeScript 中文手册](https://typescript.bootcss.com/tutorials/typescript-in-5-minutes.html)