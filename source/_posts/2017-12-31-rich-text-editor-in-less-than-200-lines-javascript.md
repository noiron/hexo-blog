---
layout: post
title: 不到200行 JavaScript 代码如何实现富文本编辑器
date: 2017-12-31
tags: [javascript, 编辑器]
---

前段时间在寻找一些关于富文本编辑器的资料，然后发现了这个名为 [Pell](https://github.com/jaredreich/pell) 的项目，它是一个所见即所得（WYSIWYG）的文本编辑器，虽然它的功能很简单，但是令人吃惊的是它只有 1kb 大小。而项目最核心的文件 [pell.js](https://github.com/jaredreich/pell/blob/master/src/pell.js) 只有130行，即使加上其它部分，总的 js 数量也不到200行。这引起了我的兴趣，决定看看它的源码是如何做到这一点的。

<!-- more -->

项目的主要代码在 `pell.js` 文件中，其结构很简单，主要功能的实现依赖于以下的几个部分

- `actions` 对象
- `exec()` 函数
- `init()` 函数

## Document.execCommand()

先从最简单的部分看起， `exec()` 函数只有下面三行：

```js
export const exec = (command, value = null) => {
    document.execCommand(command, false, value);
};
```

它将 `document.execCommand()` 进行了一个简单的包装，[Document.execCommand()](https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand) 就是这个编辑器的核心，其语法如下

```js
bool = document.execCommand(aCommandName, aShowDefaultUI, aValueArgument)
```

- `aCommandName` 是表示想执行的命令的字符串，比如：加粗 'bold'，创建链接 'createLink'，改变字体大小 'fontSize' 等等
- `aShowDefaultUI` 是否显示默认的用户界面
- `aValueArgument` 有些命令需要额外的输入，如插入图片、链接时需要给出地址

注：经过我的试验，在 Chrome 下改变 aShowDefaultUI 的值并未发现影响，[这个 stackoverflow 的问题](https://stackoverflow.com/questions/38188015/what-is-the-the-default-user-interface-referred-to-by-the-ashowdefaultui-param)中提到这是一个来自于旧版 IE 的参数，所以这里设置为默认的 false 即可。


## actions 对象

文件中定义了一个名为 `actions` 的对象，对应的是下图工具栏上的这一行按钮， `actions` 中的每个子对象都保存了一个按钮的属性。

![2017-12-29-pell-editor-toobar.png](/asset/images/2017-12-29-pell-editor-toobar.png)

部分代码：

```js
const actions = {
    bold: {
        icon: '<b>B</b>',
        title: 'Bold',
        result: () => exec('bold')
    },
    italic: {
        icon: '<i>I</i>',
        title: 'Italic',
        result: () => exec('italic')
    },
    underline: {
        icon: '<u>U</u>',
        title: 'Underline',
        result: () => exec('underline')
    },
    // …
}
```

这段代码中显示了名为 `bold`，`italic`，`underline` 的三个对象属性，对应于工具栏中前方的**加粗**、*斜体*、下划线按钮，可以看出它们的结构是相同的，都有下列三个属性：

- `icon`: 如何在工具栏中显示
- `title`: 就是 title 啦
- `result`: 一个函数，会赋给按钮作为点击事件，调用之前所提到的 `exec()` 函数来对文本进行操作

现在已有了 `actions` 对象，那么如何使用它呢？这就要看看 `init()` 函数了，它会根据一定的规则从 `actions` 对象中选出元素组成一个数组，数组的每一项都会生成一个按钮。下面代码中的 `settings.actions` 即为此数组，其中的每个元素都对应一个显示在工具栏上的按钮。`settings.actions` 的生成规则会在后面进行解释。

```js
// pell.js 中的 init() 函数
settings.actions.forEach(action => {
    // 新建一个按钮元素
    const button = document.createElement('button')
    // 给按钮加上 css 样式
    button.className = settings.classes.button
    // 把 icon 属性作为内容显示出来
    button.innerHTML = action.icon
    button.title = action.title
    // 把 result 属性赋给按钮作为点击事件
    button.onclick = action.result
    // 将创建的按钮添加到工具栏上
    actionbar.appendChild(button)
})
```

这样数组中的每个元素就都生成了一个工具栏上的按钮了。 


## init() 初始化函数

想使用 pell 编辑器时，只要调用 `init()` 函数来初始化一个编辑器即可。它接收一个 `setting` 对象作为参数，其中包含这样的一些属性：

- `element`: 编辑器的 DOM 元素
- `styleWithCSS`: 设置为 true 时，将会用 `<span style="font-weight: bold;"></span>` 代替 `<b></b>`
- `actions`
- `onChange`

其中最重要的是 `actions`，它是一个数组，包含了你想在工具栏显示的按钮列表。

`actions` 数组中可以有这几种元素：
- 一个字符串
- 一个有 `name` 属性的对象
- 一个对象，没有 `name` 属性，但有生成一个按钮的必需属性 `icon`，`result` 等

```js
actions: [
  'bold',
  'underline',
  'italic',
  {
    name: 'image',
    result: () => {
      const url = window.prompt('Enter the image URL')
      if (url) window.pell.exec('insertImage', ensureHTTP(url))
    }
  },
  // ...
]
```

在 `init()` 函数中会把这个 `actions`参数 和 pell.js 中定义的 `actions`对象组合起来，可以将 `actions` 对象当作一个默认设置，看以下代码：

```js
// pell.js 中的 init() 函数
settings.actions = settings.actions
    ? settings.actions.map(action => {

        if (typeof action === 'string') return actions[action]

        // 如果参数中传入的 action 已经在默认设置中存在，用传入的参数覆盖默认设置
        else if (actions[action.name]) {
            return { ...actions[action.name], ...action }
        }

        return action
    })
    : Object.keys(actions).map(action => actions[action])
```

如果参数对象 `setting` 中不包含 `actions` 数组,  则会默认使用之前定义的 `actions` 对象来初始化。


init() 函数里还有一个重要的部分，就是创建一个可编辑区域，这里创建了一个 `div` 元素，将其 `contentEditable` 属性设为 `true`，从而可以在这里使用之前提到的 `document.execCommand()` 命令了。

```js
// 创建编辑区域的元素
settings.element.content = document.createElement('div')
// 让 div 成为可编辑状态
settings.element.content.contentEditable = true
settings.element.content.className = settings.classes.content
// 当用户输入时，更新页面的相应部分
settings.element.content.oninput = event => 
    settings.onChange(event.target.innerHTML)
settings.element.content.onkeydown = preventTab
settings.element.appendChild(settings.element.content)
```


## 流程整理

最后以“插入链接”为例来梳理下整个编辑器的流程：

一、在调用 `init()` 函数时，在参数对象的 `action` 数组中加入以下一项

```js
{
    name: 'link',
    result: () => {
        const url = window.prompt('Enter the link URL')
        if (url) window.pell.exec('createLink', ensureHTTP(url))
    }
}
```


二、在 `init()` 的运行过程中，会检查已定义的 `actions` 对象中是否有 `link` 这个属性。经检查属性确实存在

```js
link: {
    icon: '&#128279;',
    title: 'Link',
    result: () => {
        const url = window.prompt('Enter the link URL')
        if (url) exec('createLink', url)
    }
}
```

因为传入的参数中有 `result` 这一项，所以用传入的 `result` 来代替 `link` 对象中的默认值，然后将修改过的 `link` 对象放入 `settings.actions` 数组中。

三、对 `settings.actions` 数组进行一次迭代来生成工具栏，`link` 对象作为其中的一项生成了一个“插入链接”的按钮。`result` 属性成为其点击事件。

四、点击“插入链接”的按钮后，会让你输入一个 url，然后调用 `exec('createLink', url)` 在编辑区域插入该链接。

编辑器其它按钮的功能流程也类似。


这样 Pell 编辑器的大部分内容就讲解完毕了，剩余部分还需要自己去看源码。毕竟项目的代码不长，以此作为文本编辑器的入门倒不错。


---

2017年的最后一篇文章了，再见，2017。

[本文原地址](http://www.wukai.me/2017/12/31/rich-text-editor-in-less-than-200-lines-javascript/)
