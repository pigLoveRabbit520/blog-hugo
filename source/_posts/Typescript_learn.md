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

<!-- more -->

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
可以看到，编译好后的就是平常的js代码，TS的一个优点在于类型的声明，这样在编译期bug就能及早发现（当然还有其他好处，例如**泛型**，**Enum**，**装饰器**）。  
注意： **TypeScript** 只会在编译时对类型进行静态检查，如果发现有错误，编译的时候就会报错。而在运行时，与普通的 JavaScript 文件一样，不会对类型进行检查。

## 复杂点
构建一个TypeScript的项目就需要`tsconfig.json`文件了。如果一个目录下存在一个`tsconfig.json`文件，那么它意味着这个目录是TypeScript项目的根目录。  

创建一个简单的项目，目录结构如下：
```
MyProj
├── src
│   ├── index.ts
│   ├── person.ts
│   ├── animal.js
├── package.json
├── tsconfig.json
├── .eslintrc.js  
```
tsconfig.json文件  
```
{
  "compilerOptions": {
    "target": "ES2018", // 指定ECMAScript目标版本
    "module": "commonjs",
    "moduleResolution": "node",
    "experimentalDecorators": true, // 开启装饰器
    "strict": true, // 启用所有严格类型检查选项
    "noImplicitAny": false,
    "removeComments": true, // 移除注释
    "sourceMap": false,
    "rootDir": "src", // Default: The longest common path of all non-declaration input files.
    "outDir": "dist", // 编译输出目录
    "allowJs": true // 允许JS文件混合
  },
  "include": ["src/**/*"], // 指定要编译文件
  "exclude": ["node_modules"] // 指定要排除的编译文件
}
```
**person.ts**
```
export class Person {
  private name: string;

  private age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  getName() {
    return this.name;
  }
}

export interface Payload {
  title: string;
  description: string;
}

```
**index.ts**
```
import { Person, Payload } from './person';
import { Animal } from './animal';

const p = new Person('sala', 12);

const data: Payload = { title: 'One', description: 'happy...' };

type StringOrNumber = string | number;

function getString(n: StringOrNumber): string {
  if (typeof n === 'string') {
    return n;
  } else {
    return n.toString();
  }
}

const animal = new Animal('Kitty');
```
**animal.js**
```
export class Animal {
  constructor(name) {
    this.name = name;
  }

  sayHi() {
    return 'hello!';
  }
}
```

**package.json**
```
{
  "name": "ts_learn_project",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "dependencies": {},
  "devDependencies": {
    "@typescript-eslint/eslint-plugin": "^4.22.0",
    "@typescript-eslint/parser": "^4.22.0",
    "eslint": "^7.24.0",
    "typescript": "^4.2.4",
    "mwts": "^1.0.5"
  },
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "eslint": "eslint src --ext .ts",
    "build": "tsc"
  },
  "keywords": [
    "typescript"
  ],
  "author": "salamander",
  "license": "ISC"
}

```
**.eslintrc.js**  
```
module.exports = {
  parser: "@typescript-eslint/parser",
  plugins: ["@typescript-eslint"],
  rules: {
    // 禁止使用 var
    "no-var": "error",
    // 优先使用 interface 而不是 type
    "@typescript-eslint/consistent-type-definitions": ["error", "interface"],
  },
};

```
2019 年 1 月，TypeScirpt 官方决定全面采用 ESLint 作为代码检查的工具，并创建了一个新项目 [typescript-eslint](https://github.com/typescript-eslint/typescript-eslint/blob/master/packages/parser)，提供了 TypeScript 文件的解析器 @typescript-eslint/parser 和相关的配置选项 @typescript-eslint/eslint-plugin 等。























参考：
* [TypeScript 中文手册](https://typescript.bootcss.com/tutorials/typescript-in-5-minutes.html)
* [tsconfig](https://www.typescriptlang.org/tsconfig)