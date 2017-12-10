---
layout: post
title: 塔防游戏中的敌人如何沿路径前进 (JavaScript 实现)
date: 2017-12-09
tags: [javascript, 游戏开发]
---

如果开发一个塔防游戏，很自然的会遇上这么两个名字很像的问题：

- **Path-finding**: 如果知道起点和终点，如何在其间找到一条路径
- **Path-following**: 已知从起点到终点的路径，物体如何才能沿着它行进

本文将要讨论的是第二个问题 **path following**，给定一条路径，看物体如何沿着它从起点运行至终点。为了方便描述，接下来的内容中，用单词 [Boid](https://en.wikipedia.org/wiki/Boids) 来表示行进的物体或塔防中的敌人。

接下来会用一种简单的方法来解决这一问题，最终完成的代码库可见 GitHub: [boid-path-following](https://github.com/noiron/boid-path-following)，repo 的多个分支对应了文中的不同步骤。

<!-- more -->

## 准备工作

先来看看如何标识出画面中的位置，首先画面被一系列的横纵线分成了许多网格，对于地图范围内的一个点，它会有自己的像素坐标 (x, y)，同时它所处的格子也有自己的坐标 (col, row) 或 (xIndex, yIndex)，表示所处的列和行。

![](/asset/images/2017-12-09-path-following-map.png)

为了区分，下文中提到**像素坐标**即为用像素表示的坐标，**网格坐标**表示点在网格中的列和行。

在这种表示方法下，还需要一个工具函数 `index2Px(col, row)`，用于计算格子中心的像素坐标。

接下来给出路径的坐标，路径是如下的一个二维数组：

```javascript
    const path = [[0, 1], [COLS - 4, 1], [COLS - 4, 4], [6, 4], [6, 7], /* 部分省略 */]
```

每一项都是路径上一个点的网格坐标，将这些点用直线连接起来后就得到了 boid 行进的路径。我们的目标就是要让 boid 能够从路径第一个坐标移动至最后一个坐标。


## Boid 如何沿直线前进

先考虑最简单的问题，如何让 Boid 沿着一条直线行进。

物体的移动需要位置和速度，为了表示其像素坐标，`boid` 需要 `x`, `y` 属性；其速度需要 `speed` 属性，同时还需要一个 `angle`，以便计算出速度在两个方向上的分量 `vx`, `vy`。

动画效果的实现需要用 `requestAnimationFrame` 函数，每一秒为60帧，每一帧中都会执行一次循环，在其中改变位置：

    下一时刻的位置 = 当前时刻的位置 + 速度

```javascript
    // 示意代码
    // Boid 类的 step() 方法
    step() {
        const speed = this.speed;
        const angle = Math.PI / 2;

        this.vx = Math.cos(angle) * speed;
        this.vy = Math.sin(angle) * speed;

        // 如果 vx, vy 不变化，则会沿一条直线前进
        this.x += this.vx;
        this.y += this.vy; 
    }
```

在每一个循环中，boid 的位置都会发生变化，在新的位置上将其画出即可看到 boid 沿直线运动的效果。

这一部分可在示例代码库的 `demo01/go-straight` 分支上查看：

    git checkout demo01/go-straight
    npm run demo01

现在 boid 已经动起来了，但是却没法停止，这就是我们接下来需要考虑的问题。

## 如何让 boid 在目标点处停止

要让 boid 能够知道自己到达了目标点，则在每一次循环过程中，需要计算出此刻离目标点的距离分量 `dx`，`dy`，据此算出距离 `dist`，将其与速度 `speed` 进行比较。如果 `dist > speed`，说明物体离目标点还挺远，继续将速度加到位置上即可。反之则表明物体将要到达终点，此时若直接加上速度，boid 可能会越过目标点，因此需要一点不同的处理。

```javascript
    // 示意代码
    step() {
        if (reachDest) {
            // 已到达终点，可根据实际需要进行操作            
        }

        const speed = this.speed;

        // 与目标点的距离
        this.dx = target.x - this.x;
        this.dy = target.y - this.y;
        this.dist = Math.sqrt(this.dx * this.dx + this.dy * this.dy);
        this.angle = Math.atan2(this.dy, this.dx);

        // 速度分量
        this.vx = Math.cos(this.angle) * speed;
        this.vy = Math.sin(this.angle) * speed;

        if (this.dist > speed) {
            this.x += this.vx;
            this.y += this.vy; 
        } else {
            // 当前时刻的位置加上速度后超过了当前目标点
            // 物体下一时刻将处于当前目标点的位置
            this.x = target.x;
            this.y = target.y;
            this.reachDest = true;
        }
    }
```

这一部分可在示例代码库的 `demo01/stop` 分支上查看：

    git checkout demo01/stop
    npm run demo01

此时，到达了终点的 boid 被清除而不再显示。

## 如何让 boid 能够转向

前面叙述中为了简化，路径中只有起点和终点，所以 boid 没有机会转向，那当路径变复杂了之后，boid 该如何运动？

前面已经提到过，`path` 是一个记录了路径网格坐标的数组，boid 会从中取一个坐标作为自己的当前目标点，然后一直向前行进，到达了这个目标点之后，它会从 `path` 数组中取出下一个坐标，继续移动至该位置。循环以上过程，直到 boid 到达 `path` 中的最后一个坐标。

上面的代码中，我们的目标点 `target` 固定为 `path` 的最后一个坐标，而现在每一次转向时 `target` 都会变化，所以加入这样的两个变量：

- `waypoint` 表示**当前**目标点的索引
- `angleFlag` 记录是否需要转向。

```javascript

// Boid 的 step() 中的部分示意代码

/* 每次转向后目标点需要重新计算 */
const waypoint = path[this.waypoint];   // 当前目标点的网格坐标
const target = index2Px(...waypoint);   // 当前目标点的像素坐标

// ...

// 判断是否需要转向，如果需要转向，则重新计算角度
if (this.angleFlag) {
    this.angle = Math.atan2(this.dy, this.dx);
    this.angleFlag = 0;
}

// 每次到达一个目标点之后，都要检查是否为终点
if (this.waypoint + 1 >= path.length) {
    // 到达终点
    this.reachDest = true;
} else {
    this.waypoint++;
    this.angleFlag = 1;
}
```

这一部分可在示例代码库的 `demo01/steering` 分支上查看：

    git checkout demo01/steering
    npm run demo01

结果可见下图：
![](/asset/images/2017-12-09-path-following-1.png)

到此为止，这种 boid 沿路径行进的方法已经讲解完毕了。建议读者查看一下 repo 中的代码，自己修改部分代码，比如更改路径，看结果会有何不同。

## 其它的方法

这一种方法中的确实现了沿路径移动的效果，但是有点儿单调，boid 只能在路径的中轴线上移动，而且它们之间也没有交互的效果。[The Nature of Code](http://natureofcode.com/book/) 这本书的第六章 [Autonomous Agents](http://natureofcode.com/book/chapter-6-autonomous-agents/) 中介绍了另一种稍微复杂的方法来实现 path following。

我之前参考他人的代码实现了[这种方法的一个演示版本](http://www.wukai.me/html-demo/path-following/)，其代码在[此处](https://github.com/noiron/html-demo/tree/master/path-following)。

(也许之后会补一篇博客来介绍 *The Nature of Code* 中的实现，但谁知道会不会写呢🤔)

## 结语

最后，[我最近在写的这个塔防游戏](https://github.com/noiron/tower-defense-js)中就使用了本文介绍的 path following 方法。虽然游戏还没完成，但点进去看看再给个 star 又不费电😀。

---

[本文地址：塔防游戏中的敌人如何沿路径前进 (JavaScript 实现)](http://www.wukai.me/2017/12/09/boid-path-following/)
