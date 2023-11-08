---
title: Puppeteer使用例子
author: Salamander
tags:
  - puppeteer
  - Nodejs
categories: []
date: 2021-03-05 09:00:00
---
![Puppeteer](https://s3.ax1x.com/2021/03/05/6eGwVJ.png)  
这篇文章很简单呢，就是记录一下用**Puppeteer**的一些snippet。

<!-- more -->


## 访问网站后截图
```CSharp
const puppeteer = require('puppeteer');
const url = 'https://segmentfault.com';

(async() => {
    const browser = await puppeteer.launch({
        headless: true,
        args: ["--no-sandbox", "--single-process"],
    });
    const page = await browser.newPage();
    await page.setViewport({ width: 1920, height: 1080 });
    // ‘networkidle2’ means that there are no more than 2 active requests open. 
    // This is a good setting because for some websites (e.g. websites using websockets) there will always be connections open
    await page.goto(url, {
        waitUntil: 'networkidle2',
        timeout: 1000 * 60 * 5, // 毫秒 超时参数需要加上，有时候网络不好，会导致等着
    });
    await page.screenshot({path: './data/website.png', type: 'png'});
    page.close();
    browser.close();
})();
```