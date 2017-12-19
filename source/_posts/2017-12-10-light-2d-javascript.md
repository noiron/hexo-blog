---
layout: post
title: 用 JavaScript 画光：基础
date: 2017-12-10
tags: [javascript, 计算机图形学]
---

> Any application that **can** be written in JavaScript, **will** eventually be written in JavaScript.
> -- Atwood's Law

本文来源于我在看了 Milo Yip 在知乎专栏里的这篇文章：[《用 C 语言画光（一）：基础》](https://zhuanlan.zhihu.com/p/30745861)之后的一个想法，能不能将原文中 C 语言版本程序改成 JavaScript 版本的。动手之后发现出乎意料的顺利，我只需要把 C 语言中变量的类型通通去掉就可以了😀，Amazing！

<!-- more -->

最终结果可见此 CodePen：https://codepen.io/noiron/pen/aVgYMB?editors=1010

<p data-height="400" data-theme-id="0" data-slug-hash="aVgYMB" data-default-tab="result" data-user="noiron" data-embed-version="2" data-pen-title="aVgYMB" class="codepen">See the Pen <a href="https://codepen.io/noiron/pen/aVgYMB/">aVgYMB</a> by wu kai (<a href="https://codepen.io/noiron">@noiron</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>

在本文中，我主要解释一下 JavaScript 如何将图像输出，以及我对这个画光程序的一点理解。更多有关图形学原理部分的内容，建议还是看 Milo Yip 的原文。

## 如何输出图像

Milo Yip 在他的系列文章中使用了一个自己写的 [svpng()](https://zhuanlan.zhihu.com/p/26525083) 函数，能够根据得到的图形数据生成 `png` 格式的图片。而使用 JavaScript 可以方便地在 `canvas` 元素上绘制出图形。

为了能够记录下图片的信息，需要记录每一个像素点的 `RGB` 值，对于一张宽度为 W，高度为 H 的图片，其像素点数量为 `W * H`，而每个像素点分别用三个数来表示其 R、G、B 值，所以记录下整张图片的数据，需要一个长度为 `W * H * 3` 的数组。如果图片带有 alpha 通道，需要记录 `RGBA` 值，则数组长度为 `W * H * 4`。这里有一个可以简化的地方，因为绘制的是一张黑白的图片，对于黑/白/灰色来说 R = G = B，所以用长度 `W * H` 的数组即可。

假设我们现在已经有了一个记录图片信息的数组 `p`，那么如何将其显示出来？这里需要用到 `getImageData`, `putImageData` 方法。

可以利用 `getImageData()` 方法来获得 ImageData 对象，从中得到图像的像素点。

```javascript
const ctx = canvas.getContext('2d')
// 获取 ImageData 对象
const imageData = ctx.getImageData(x, y, width, height)
```

`ImageData` 对象的 `data` 属性是一个数组，包含有每个像素点的 `RGBA`，其总长度为 `W * H * 4`。所以我们将记录图片信息的数组 `p` 中的值依序赋给 `data`，再利用 `putImageData` 方法即可将图片绘制到 canvas 上了。

```javascript
function processImageData(imageData, p) {
    const data = imageData.data;
    for (let i = 0; i < data.length; i += 4) {
        const value = p[i / 4];
        data[i] = value;
        data[i + 1] = value;
        data[i + 2] = value;
        data[i + 3] = 255;
    }
}

processImageData(imageData, p);
ctx.putImageData(imageData, 0, 0);
```

## 如何获得一个点的光照强度

现在我们考虑的是单色光，RGB 中的三个值是相等的，当光照越强时，RGB 值越大，图像的颜色也越白。

坐标为 (x, y) 的一个点，它获得的光来自于各个方向上的光的叠加，即是一个对角度的积分：

$$F\left( x,y\right) =\int ^{2\pi }_{0}L\left( x,y,\theta \right) d\theta$$

其中 $L\left( x,y,\theta \right)$ 代表在二维坐标 (x, y) 在 $\theta$ 方向有多少光经过。

由于无法直接计算出这个积分的值，需要用**蒙特卡罗积分法**来进行采样。利用 N 个方向的采样平均值作为这一点的光强。

那么 (x, y) 点在 $\theta$ 方向上能获得多少光照？我们现在只有一个处于画面中央的圆形光源，可考虑从 (x, y) 为起点的一条线段，如果它足够长，那只有两种可能性：
 
 - 终结于光源的表面，则 (x, y) 点在 $\theta$ 方向能获得光照
 - 与光源无交点，则此方向上无光照

但我们需要对这条线段的长度加以限制，所以逐步加长线段的长度，如果线段终点在光源的表面或内部，则获得光照。当步数达到 `MAX_STEP` 或距离达到 `MAX_DISTANCE`，停止计算，在此方向上获得的光照为0。

这里需要利用**带符号距离场（signed distance field, SDF）**来表示出当前的点与场景的最近距离，每次步进此距离能保证不会进入光源的内部。如下图中，每个圆的半径均为圆心和图中形状的最近距离，则按 P0 -> P1 -> P2 -> ... 的顺序前进能保证不会和图中的形状相交。 

![sphere-tracing](/asset/images/2017-12-10-sphere-tracing.jpg)
（图源：https://developer.nvidia.com/gpugems/GPUGems2/gpugems2_chapter08.html）

此即原文中提到的**光线步进（ray marching）**方法（又称为**球体追踪／sphere tracing**）。


## JavaScript 的实现

利用 `sample()` 函数计算并保存所有坐标点的光照：

```javascript
const p = [];
for (let y = 0, i = 0; y < HEIGHT; y++) {
    for (let x = 0; x < WIDTH; x++) {
        // x / W, y / H 其值被限制在 [0, 1] 之间
        p[i++] = Math.floor(Math.min(sample(x / WIDTH, y / HEIGHT) * 255, 255));
    }
}
```

利用蒙特卡罗积分法进行 N 次采样取平均值获得 (x, y) 处的光照强度，其中的 `trace()` 函数代表的是从 $\theta$ 方向获取的光强。

```js
function sample(x, y) {
    let sum = 0;
    for (let i = 0; i < N; i++) {
        // 以下为三种不同的采样方式
        // const theta = Math.PI * 2 * Math.random();           // 随机采样
        // const theta = Math.PI * 2 * i / N;                   // 分层采样（stratified sampling）
        const theta = Math.PI * 2 * (i + Math.random()) / N;    // 抖动采样（jittered sampling）

        // trace() 所返回的值是点 (x, y) 从 theta 方向获取的光
        sum += trace(x, y, Math.cos(theta), Math.sin(theta));
    }
    return sum / N;
}
```

`circleSDF` 为带符号距离场（signed distance field, SDF），值为负时，表示在光源的内部。

```js
function circleSDF(x, y, cx, cy, r) {
    const ux = x - cx;
    const uy = y - cy;
    return Math.sqrt(ux * ux + uy * uy) - r;
}
```

最后就是 `trace()` 方法，用**光线步进**来计算出 (ox, oy) 沿单位向量 (dx, dy) 方向上获得的光照。

```js
function trace(ox, oy, dx, dy) {
    const MAX_STEP = 10;
    const MAX_DISTANCE = 2;
    const EPSILON = 1e-6;

    let t = 0.0;    // t 为步进的距离
    for (let i = 0; i < MAX_STEP && t < MAX_DISTANCE; i++) {
        // 光源中心为 (sourceX, sourceY) 
        // 沿单位向量 (dx, dy) 方向前进，t 表示前进的距离
        const sd = circleSDF(ox + dx * t, oy + dy * t, sourceX, sourceY, 0.1);

        // 此时已到达发光的圆形表面
        if (sd < EPSILON) {
            return 2.0;
        }
        // 继续增加步进的距离
        t += sd;
    }
    return 0.0;
}
```

最后我还在代码添加了一个点击事件，可以改变光源位置来查看不同的效果。


## 参考资料

[Canvas 操作图像像素](https://github.com/YIXUNFE/blog/issues/12)
