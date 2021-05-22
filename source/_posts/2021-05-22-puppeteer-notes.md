---
layout: post
title: Puppeteer 的使用记录
date: 2021-05-22
tags: [puppeteer, node.js, 爬虫]
---

**Headless Browser** 指的是没有图形用户界面（GUI）而是由程序控制的浏览器。**Puppeteer** 就是由 Google 推出的一种 headless browser。从 [Puppeteer 的官方文档](https://github.com/puppeteer/puppeteer)中可以看到它能做的事有很多：

> - Generate screenshots and PDFs of pages.
> - Crawl a SPA (Single-Page Application) and generate pre-rendered content (i.e. "SSR" (Server-Side Rendering)).
> - Automate form submission, UI testing, keyboard input, etc.
> ...

几乎所有能手动在 Chrome 进行的操作现在都可以用 Puppeteer 来完成。

我之前在工作中就用到了 Puppeteer 做了一些页面爬取的工作，这里把使用到或者玩过的一些功能做了下总结和记录。这篇文章并不是一个全面的 Puppeteer 的使用教程，毕竟 Puppeteer 的大部分 API 我也没有使用过🙂。

<!-- more -->

## Puppeteer 的安装

Puppeteer 的使用就是如同一个普通的 npm package 一样，使用 `npm init` 新建一个项目后，使用 `npm install puppeteer` 即可。如果安装过程中卡在了下载 Chromium 的过程中，可以考虑使用 `puppeteer-core` 来代替，具体可见

[puppeteer-vs-puppeteer-core](https://github.com/puppeteer/puppeteer/blob/main/docs/api.md#puppeteer-vs-puppeteer-core)

## 打开页面

先由最基础的打开页面功能开始，新建一个 `index.js` 文件，写入下方的代码。

```js
const puppeteer = require('puppeteer');
const pageUrl = 'https://github.com';

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto(pageUrl, { waitUntil: 'networkidle0' });
  await page.close();
  await browser.close();
})();
```

现在运行 `node index.js`，如果成功运行，你将会看到什么也没有发生😕。这是因为，Puppeteer 默认是在 `headless` 模式下运行的，为了能看到确实打开了页面，可以在打开浏览器时加入参数 `{ headless: false }`。

```js
const puppeteer = require('puppeteer');
const pageUrl = 'https://github.com';

(async () => {
  // 这里将 headless 设置了为 false，表示打开图形界面
  const browser = await puppeteer.launch({ headless: false });
  const page = await browser.newPage();
  await page.goto(pageUrl, { waitUntil: 'networkidle0' });
  await page.close();
  await browser.close();
})();
```

一个典型的流程如下：打开浏览器 -> 打开新页面 -> 去往指定页面 -> 关闭页面（可省略） -> 关闭浏览器。

### Timeout 问题的解决

运行上面的代码，可能会出现 `TimeoutError` 的出错提示，这是因为访问指定页面的速度较慢导致了超时。你可以将 `pageUrl` 换成任意国内地址重试。

默认情况下 Puppeteer 的超时时间为 30s，如果在 30s 之内没能打开页面，会有 timeout 的出错提示。
```
UnhandledPromiseRejectionWarning: TimeoutError: Navigation timeout of 30000 ms exceeded
```

可以通过如下设置解决：

```js
const page = await browser.newPage();
// 将超时时间改为了 60s，如果参数为0，则代表不做限制
page.setDefaultNavigationTimeout(60000);
await page.goto(pageUrl, { waitUntil: 'networkidle0' });
```

## 操作页面

### 截图

Puppeteer 提供了 `screenshot` API，可以将页面截图保存下来。运行下面的代码将会在 js 文件所在的路径下保存一张名为 `screenshot.png` 的图片文件。这里有一点需要注意，在之前的代码中，我们没有设置窗口的大小，Puppeteer 默认打开的是 `800*600`尺寸的窗口，截图的大小也为 `800*600`（即使是在 `headless` 模式下也是如此）。可以通过 `setViewport` 来改变窗口的大小。

```js
const path = require('path');
const puppeteer = require('puppeteer');
const pageUrl = 'https://github.com';

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto(pageUrl, { waitUntil: 'networkidle0' });
  // 设置窗口的大小，可以注释下方的一行查看截图的区别
  await page.setViewport({ width: 1000, height: 500 });
  await page.screenshot({
    path: path.join(__dirname, 'screenshot.png'),
  });
  await page.close();
  await browser.close();
})();
```

### 生成 PDF

同样也可以将页面保存成一个 PDF 文件。

```js
const path = require('path');
const puppeteer = require('puppeteer');
const pageUrl = 'https://github.com';

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto(pageUrl, { waitUntil: 'networkidle0' });
  await page.pdf({ 
    path: path.join(__dirname, 'page-pdf.pdf'), 
    format: 'A4'
  });
  await page.close();
  await browser.close();
})();
```

注意一点，保存为 PDF 的时候，必须在 `headless` 模式下进行。

## 模拟设备

平时我们浏览的网站可能针对不同的设备会有不同的展示效果，区分移动端和PC端、Android 或 iOS 等。Puppeteer 就提供了模拟设备的功能。

### 使用已有的设备列表

可以使用 `page.emulate(options)` 来模拟设备，可以在这个文件中看到支持的设备列表：[puppeteer/DeviceDescriptors.ts at main · puppeteer/puppeteer · GitHub](https://github.com/puppeteer/puppeteer/blob/main/src/common/DeviceDescriptors.ts)

```js
const puppeteer = require('puppeteer');
const pageUrl = 'https://github.com';
// 这里将模拟一台 iPhone XR 设备
const iPhone = puppeteer.devices['iPhone XR'];

(async () => {
  const browser = await puppeteer.launch({ headless: false });
  const page = await browser.newPage();
  await page.emulate(iPhone);
  await page.goto(pageUrl, { waitUntil: 'networkidle0' });
  await browser.close();
})();
```

### 添加自定义的 UserAgent

也可以添加自己的 UserAgent 的方式来模拟列表中没有的设备。下面的代码中以 iPad Pro 为例，说明如何设置 UserAgent（其实 iPad Pro 是存在于上面的设备列表中的，这里只是举了个例子）。

```js
const puppeteer = require('puppeteer');
const pageUrl = 'https://github.com';

(async () => {
  const browser = await puppeteer.launch({ headless: false });
  const page = await browser.newPage();
  await page.setViewport({ width: 1024, height: 1366 });
  await page.setUserAgent(
    'Mozilla/5.0 (iPad; CPU OS 11_0 like Mac OS X) AppleWebKit/604.1.34 (KHTML, like Gecko) Version/11.0 Mobile/15A5341f Safari/604.1',
    );
  await page.goto(pageUrl, { waitUntil: 'networkidle0' });
  await browser.close();
})();
```

## 监听 response

考虑一下这样的场景，如何获取到页面上的所有的图片。当然我们可以拿到整个页面，然后使用 `document.querySelector()` 筛选出所有的 `img` 标签。这样会有一点小问题，如果图片是以 CSS  的 `background-image` 形式引入的，查找 `img` 标签是无法找到的。如果图片地址是由 js 动态改变的，可能也会缺失部分图片。

另一种可选的方式是通过监听 `response` 的方式来处理。可以认为在 Chrome 开发工具的 Network Tab 下能看到的每一个 response 都可以在这里拿到。

```js
const path = require('path');
const puppeteer = require('puppeteer');
const pageUrl = 'https://github.com';

(async () => {
  const browser = await puppeteer.launch({ headless: false });
  const page = await browser.newPage();

  page.on('response', async (response) => {
    const request = response.request();
    const resourceType = request.resourceType();
    const contentType = response.headers()['content-type'];

    if (resourceType === 'image') {
      const url = response.url();
      console.log(contentType, url);
    }
  });

  await page.goto(pageUrl, { waitUntil: 'networkidle0' });
  await browser.close();
})();
```

通过 `response.headers` 可以获取响应的 header 信息。如果有需要也可将响应保存成本地的文件。

---

未完待续