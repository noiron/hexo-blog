---
layout: post
title: (译)用神经网络实现能够自主避让障碍的生物
date:   2016-03-01
tags: [javascript, 神经网络]
categories: 
---

这篇文章是我去年看到的一个很有趣的项目，还试着模仿它的代码写一个类似的项目出来，不过一直没有完成。这里把原作者的一篇相关博客翻译过来，说不定有更多的人对此感兴趣。

- 项目演示：[原作者用JavaScript实现的demo](http://otoro.net/planks/)
- 原文地址：[creatures avoiding planks](http://blog.otoro.net/2015/05/07/creatures-avoiding-planks/)
- 原作者的twitter地址：[hardmaru's twitter](https://twitter.com/hardmaru)
- 原作者的另一个实验：[生成伪汉字的实验](http://otoro.net/kanji/)
- 译文地址：[用神经网络实现能够自主避让障碍的生物](http://noiron.github.io/2016/03/01/creatures-avoiding-planks/)
- 翻译：吴锴（noiron）

**注意：原文中引用了一些墙外链接，请配合梯子食用**

以下为译文内容：
***************

![](/asset/images/2016-03-01-planks_blog.png)
**具有神经网络大脑的agent为了生存而进化**

[这里是javascript的演示](http://otoro.net/planks/)

最近我看到了一个模拟[视频](https://www.youtube.com/watch?v=fmSTNu0Zjh8)，展示了使用进化技术来训练agent，让它们能够避开移动的障碍物。[这里使用的方法](https://www.reddit.com/r/gamedev/comments/31idqb/im_building_evolution_based_ai_toolkit_to/)似乎是[NEAT](http://www.cs.ucf.edu/~kstanley/neat.html)算法的变种。NEAT算法用于神经网络拓扑结构的进化，使其能够正确完成特定的任务。它是为了[Unity 5](http://en.wikipedia.org/wiki/Unity_(game_engine)游戏引擎而写的，想被整合进综合的游戏AI中，这引起了我的兴趣。

这个结果很酷而且动起来似乎很优雅，所以我想试着用javascript实现一个类似的demo，这样就可以在浏览器中运行了。在玩了一下基本的实现后，我发现对于障碍躲避问题，一个简单的神经网络就可以高效的完成了，甚至于一个完全连接的神经网络都算是杀鸡用牛刀了。最后我也意识到就算是一个简单的类[感知器（perceptron）](http://upload.wikimedia.org/wikipedia/commons/3/31/Perceptron.svg)网络都能够很好地完成任务，而在之前提到的障碍躲避demo中，隐藏单元也并不能够增加多少好处。

![perceptron](/asset/images/2016-03-01-perceptron.png)
**简单的感知器图解（维基百科）**

因为agent可以在探测到障碍物的瞬间自由地向任意方向移动，而且agent之间不会产生相互作用，因此这个任务变得太简单了，使用NEAT算法的杀伤力过大了。我们使用一个简单的类感知器网络（即单神经元网络），就能让agent有效地完成躲避障碍的任务了。

<!-- more -->

## 让agent的生活变得艰难，因为生活是艰难的

我想让事情有趣点，所以加了几个额外的条件让任务更困难。

1. 运动被限制在严格版本的[Reynolds转向](http://www.red3d.com/cwr/steer/)下，如[这篇论文](http://www.cs.uu.nl/docs/vakken/mcrs/papers/8.pdf)中概述的创建逼真的模拟行为这样，与之相对应的是可以向任意方向自由移动。

2. Agent可以选择向左或向右转向，但不能够直行。就像一辆没有动力的车转弯一样，它也必须在转弯时朝着所转的方向前进。我想看看agent能否发展出这样一种转向模式，即通过快速地在左转和右转之间切换从而向前移动。这样事情变得更加困难了，你可以想象驾驶一辆汽车时，为了能向前直行，只能不断地向左或向右打死方向盘。

3. Agent之间会互相碰撞，如果适应度函数是按这种方式设置的话，它们也许需要发展出一种策略来避开对方，要不然我内心的邪恶面会希望它们一起跑向木板。

我发现有了这些限制，agent就不能依靠一个简单的单神经元完成这一任务了。我仍然认为这一任务不应当依靠类似ESP或NEAT等高级的方法，我想用现成的[convnet.js](http://cs.stanford.edu/people/karpathy/convnetjs/)库来生成神经网络，所以我选择了在[neural slime volleyball](http://blog.otoro.net/2015/03/28/neural-slime-volleyball/)项目中使用过的同样超级简单的神经网络结构。下面是我的第一次尝试：

<iframe src="https://instagram.com/p/1qJiRMwiaw/embed/captioned/?v=4" width="80%"></iframe>

每一个agent的大脑由十个神经元组成，每个神经元都和所有的输入及其余的神经元相连接，也许就像真实大脑的中一个小区域的切片一样。下面是大脑的原理图：

![basic diagram of the neural network](/asset/images/2016-03-01-planks_net_8_recurrent.png)
**神经网络的基本图解——每个大脑有269个权重值**


这种网络结构被称作[全连接的递归网络（fully connected recurrent network）](http://en.wikipedia.org/wiki/Recurrent_neural_network#Fully_recurrent_network)。每一个agent都有一组将送入递归网络中的感知输入。10个神经元中的2个将控制agent的动作，其它的是用于计算和思考的“隐藏神经元单元”。所有的神经元输出信号又将送回输入层，所以每一个神经元都与其它的神经元完全连接起来了，这中间会有一帧（~1/30s）的时间延迟。

输出层的第一个神经元用于控制agent是向左还是向右转，取决于输出的信号。我们使用[双曲正切](http://mathworld.wolfram.com/HyperbolicTangent.html)函数来启动神经元，因为神经元的输出值在-1至1之间，所以做这样的一个二元选择（左/右）是很自然的。第二个神经元控制agent是移动还是保持静止，也取决于输出信号。


## 训练网络

100个初始化为随机权重值的agent被用于训练，它们会在模拟中四处跑动直到它们被一块木板杀死。在模拟的最后所有人都死了，这种感觉很糟糕，但是人固有一死……

当所有的agent在一次模拟中全都死亡了之后，我们保留在模拟中存活了最长时间的30个agent的基因用于下一代，丢弃其余的70个agent。最好的30个基因，即神经连接的权值和偏置（bias）的向量，将会进行交叉变异以传递给下一代中新产生的70个agent。这个过程（将在下面的繁殖部分解释）将会持续进行，直到最好的agent能够熟练地躲开木板并存活五分钟以上。

下面是一个运行中的进化模拟[demo](http://otoro.net/planks/training.html)，它的第一代是完全随机的网络（我将这个demo缩减为50个agent而不是100个，这样在大部分电脑甚至是智能手机上都能够运行）。起初，大部分的agent都不会试图存活下去（或者直接放弃了！），但是也许一个或两个知道该干些什么并且做的好了些，所以它们能够进入下一代并且进行繁殖。在几代过后，你会看到种群的能力增加了，它们全都开始躲避移动的木板了。我惊讶地发现，在30代后结果就不错了。如果你感兴趣，可以看一下这个[训练demo](http://otoro.net/planks/training.html)中它们是如何从开始时完全不知道如何生存进化到这一步的。注意这些结果是每十代更新一次的。因为我希望它们尽可能快的进化，而不是将进化限制在每秒30帧，所以把模拟运行到电脑能允许的最快速度，而每隔十代运行一次作为展示的模拟，因此你可以品评实时的进度——而实际的进化是在屏幕之后以光速进行的。

![](/asset/images/2016-03-01-planks_training-1024x1022.png)
**训练的草图——观看agent如何从一无所知进化而来**


Agent周围的线条是它的知觉传感器，就像视觉输入一样，它们给每一个agent提供了周围世界的感知能力。Agent感知到的物体越近则线条的强度越大。每个agent能够看到周围的8个方向，探测距离最大为其身体半径的12倍。我是这样决定信号值的，agent探测到的物体越近，信号越接近一，反之，若没有探测到物体，信号为零。如果agent看到的物体距离为6倍半径，则信号为0.25而不是0.5，这里我为了模仿[光照密度模型](http://en.wikipedia.org/wiki/Inverse-square_law)将结果进行了平方。


## 如果Wayne Gretzky在有孩子之前就遭遇了车祸会如何？

*[译者注]：Wayne Gretzky是著名的加拿大冰球运动员*

我也给上面提到的训练算法进行了一点微调。

现在的一个问题是一些表现最好的agent可能会意外死亡，即使它们很厉害，但因为模拟中agent之间太拥挤了，它也会因为纯粹运气不好而死掉。上一代的一个顶尖的躲避墙的agent，起始位置可能是在人群的边缘，在模拟开始时就会被其它的几个agent意外的推到木板上面，这就不能把它的好基因传给下一代了。

我想了一下这个问题，想出了一个主意，在进化算法中加入一个“名人堂”部分，这将会记下那些达到了最佳记录分数的agent的基因。我也修改了我的[进化库](https://github.com/hardmaru/convnetjs/tree/master/neuroevolution)来加入这个名人堂特性，以保留历来的十佳冠军。对于每一次模拟，名人堂都会加入被加入到当前的这一代。从这层意义上来说，每一代的agent都必须和历来最好的agent竞争。想象一下能够在每一个冰球赛季都和Wayne Gretzkey, Doug Gilmour及Tim Horton一起训练！

这个技术绝对极大提升了进化的结果，因为最好的基因都被强制加入到了种群中。如果这发生在现实生活中会很恐怖的（虽然我们可能很快就将达到这一步了……）。我认为我在进化计算的世界里发现了一个伟大的新技术，这是一个突破！然而，我发现名人堂技术[已经在围棋领域中被用于](http://www.cs.utexas.edu/users/nn/pages/research/ne-applications.html)神经进化了。好吧，这对我不是好消息。

我想看看如果我重新进化一遍，agent是否会表现地完全不同，于是我花了几天训练了不同的几组agent，每一组都有一千代左右。事实证明大部分的agent最终都有相似的权值模式。为了知道每个agent的大脑看起来有何不同，我将表现最好的agent的“基因代码”画成了图，为了图中的效果我将四个权值作为一组合成了一个颜色（RGBA的四个通道）。

![](/asset/images/2016-03-01-plank_boid_genes-1024x1024.png)
**每一行代表进化最成功的神经网络的权值**

可以看出表现最好的agent除了些许差别外似乎都差不多。然而，这可能会导致一个问题，即它们可能都被限制在一个局部优化状态。我可以想象在某些情境中，agent为了避开木板而趋向于发展出相同的次最优策略，而由于任务的难度的限制就不会有新的进展了。如果使用更高级的方法可以解决这一问题，但是我对现在用简单的[卷积神经进化](http://blog.otoro.net/2015/02/06/conventional-neuroevolution-cne-trainer-part-ii/)生成的递归网络已经很满意了。我在demo中使用了最好的23个基因。最终的模拟将会利用从这个训练完成的组中随机挑选出的名人堂来进行初始化。


## 结果分析

在经过约一千代的训练后，agent已经能不错地躲避障碍了。请看这个[demo](http://otoro.net/planks)中的最终结果。Agent还远不够完美，但是它们展示出的像昆虫般的行为让我觉得很有趣。我很惊讶，即使它们被限制成只能全力左转或右转，它们仍然能够完成躲避木板的任务。在demo中，每个agent有一只手，用于显示它们转的方向。没有手则表示保持静止。我也很高兴的看到当agent想朝一个方向前进时，它能在空白区域摇摆向前，看来它们通过左转和右转间的切换发展出了某种模式。

![](/asset/images/2016-03-01-planks_final_result1-1024x576.png)
**最终模拟——otoro.net/planks**

我也注意到它们倾向于和其它agent聚在一起并努力待在中间，我不确定原因。也许是因为在人群的中间，它可以等待其它agent死去从而增加自己存活的几率。我也考虑过在适应度函数中使用与种群平均值有关的*相对生命长度*，而不是绝对生命长度。也许在一个更复杂的网络中，这能够诱使agent变得邪恶且会计划杀死其他agent。


## 繁殖

![](/asset/images/2016-03-01-planks_mating.png)
**带有父母双方基因的后代**

与其每一次都等着所有的agent死亡然后创造下一代，不如让demo更有趣些，于是我给agent添加了配对和产生后代的能力！在以后这可能会形成一个可供选择的让agent进化的代内方法，而不是将每一代的种群数量固定为常数。

它工作的原理是对于每个agent，如果它们能够生存某一个随机的时间（一般为30至60秒），它们将具有繁殖的能力。如果这样的两个agent相遇了，就会产生两个后代。后代初始时体积会小一点，在约30-60秒后会长到成年大小。父代agent的“激励时钟”重置为零，必须再等30-60秒才能产生下一代……

![](/asset/images/2016-03-01-planks_crossover-1024x440.png)
**交叉算法（维基百科）**

两个后代将会继承父母双方的基因，这是通过很简单的一点[交叉方法](http://en.wikipedia.org/wiki/Crossover_%28genetic_algorithm%29)。所以第一个孩子将会从第一个父代那里继承第一段神经网络的权值，剩下的来自第二个父代。如上图中描述的一样，截断点是随机的。我还额外地给每一个孩子增加了随机变异，所以它们将具有独特性。这个随机的变异对于鼓励创新和给种群的基因库添加新元素是很重要的。

从原理上说表现越好的agent有越高的生存几率，因此它们有更高的几率产生携带其基因的后代，反之较弱的个体会在拥有孩子之前死去。虽然有时很好的个体会因为不走运而死的太早，这对于它们确实不太好——这就是生活——生活是不公平的，而且生活很艰难！

## 与agent互动——扮演上帝

当使用demo时，你可以点击屏幕的空白部分来创建新的agent加入模拟。一个新的agent的初始基因是从最近的几个试图产生后代的agent中随机选择出的，如果所有的agent都未繁殖过，那就从名人堂最初的一组中选择。

如果你点击到了木板上，将会切换到下一个视觉模式，例如，X射线模式，无边框模式，没有手的纯粹生物模式等等。我用这些模式来评估不同的设计和总体的美感。我认为X射线模式看起来很酷，而且在较慢的机器上也能运行的更快。

![](/asset/images/2016-03-01-planks_xray-300x300.png)
**X射线模式**

你也可以通过点击agent来给它们施加一个强大的推力。这模拟了一个暂时的风力效果，将它们从被点击的地方推开。

你也可以在屏幕上拖动画线来创建一块新的木板。依据所画的方向，木板会沿着其垂直方向向左或向右移动。你可以看到在你释放了一些木板去摧毁它们的文明时，agent是如何反应的。请善用这个新能力，我注意到在没有这个特性之前，用户会给小家伙们加油，但是知道了这个功能后，他们开始准备种族灭绝了！


## 设计和开发的考虑

（这一部分暂未翻译）