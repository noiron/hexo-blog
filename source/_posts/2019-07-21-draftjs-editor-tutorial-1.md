---
layout: post
title: 从插入图片功能的实现来介绍如何用 Draft.js 编写富文本编辑器
date:   2019-07-21
tags: [draft.js, 编辑器]
---


在前段时间的工作中，我遇到了一个在桌面端和移动端进行图文混排编辑的需求。虽然如果只需要编辑纯文本和图片，不一定要使用富文本编辑器来实现。但是为了以后方便扩展，比如文本会有样式要求，我还是用 Draft.js 实现了一个功能较基础的富文本编辑器。

我将代码开源在了[这个项目 draft-editor]( https://github.com/noiron/draft-editor) 中，也可以[在这里在线预览](http://www.wukai.me/draft-editor/)。本文中我将介绍一下一些关于 Draft.js 的基础知识，并由此扩展到如何在 Draft.js 编辑器中插入图片功能的实现。

<!-- more -->

## 从一个基本的编辑器开始

Draft.js 是 Facebook 推出的用于 React 的富文本编辑器框架，初始化一个最基本的 Draft.js 的代码如下：

```js
import React from ‘react’;
import ReactDOM from ‘react-dom’;
import {Editor, EditorState} from ‘draft-js’;

class MyEditor extends React.Component {
  constructor(props) {
    super(props);
    this.state = {editorState: EditorState.createEmpty()};
    this.onChange = (editorState) => this.setState({editorState});
  }
  render() {
    return (
        <Editor editorState={this.state.editorState} onChange={this.onChange} />
    );
  }
}
```

可以[在这里查看](https://codesandbox.io/s/basic-draft-editor-gzver)。这里给 `Editor` 传入一个 `editorState` 属性，并绑定一个 `onChange`事件，当发生编辑操作时，返回一个新的 `editorState`。这样我们就得到了一个可以进行基本的文本操作的编辑器了。

## Immutable.js 数据结构

在说明什么是 `EditorState` 及 Draft.js 对于数据的存储方式之前，需要简略介绍一下 **Immutables.js**。

Draft.js 是利用 [Immutable.js](https://github.com/immutable-js/immutable-js)来保存数据的，正如其名，这是一种不可变的数据结构。对于一个 Immutable 的对象，你无法修改它本身，若想修改其值，只会返回一个新的修改后的对象。将这一点应用在编辑器上，用户的每一次修改都会生成一个最新的状态快照，就很容易实现撤销功能了。

在 Draft.js 的使用过程中，可能会遇到下面这些数据结构。

**`Map`** 类似于 js 中的对象，用 `.set()` 和 `.get()` 方法进行写和读。

```
const Immutable = require(‘immutable’);
const framework = Immutable.Map({ name: 'React', age: 6 });
const newFramework = client.set('name', 'Vue');
console.log(framework.get('name'));
```

**`OrderedMap`** 混合了 `object` 和 `array` 的特点。通过使用 `orderedMap.get(‘key’)` 和
`orderedMap.set(‘key’, newValue)` 这两个方法，可以将它当成一个普通的 `object` 来使用。但和 `Map` 的不同点在于其中的 key 是按照被加入时的顺序排序的。

**`Record`** 也类似于 `Map`，但有一些不同之处。
- 一个 `record` 一旦被初始化，就不能再添加新的 key 了
- 你可以给一个 `record` 实例添加默认值

还有一点，immutable 的对象，提供了 `toJS()`方法，可将其转成普通的 js 对象，这一方法在想查看其内部内容时非常有用。

> Immutable.js 参考文章：[Immutable Data with Immutable.js | Jscrambler Blog](https://blog.jscrambler.com/immutable-data-immutable-js/)

## Draft 是如何存储数据的

### 什么是 EditorState

在创建基本的编辑器的时候，我们用到了 `EditorState`。  `EditorState` 是编辑器最顶层的状态对象，它是一个 Immutable Record  对象，保存了编辑器中全部的状态信息，包括文本状态、选中状态等。

调用  `editorState.toJS()` 可将 immutable record 转换成一个普通的 object，打印出来如下：
![](/asset/images/2019-07-21-draft-editor-01.png)

简单地来看下其中的部分内容：
- `currentContent` 是一个 `ContentState` 对象，存放的是当前编辑器中的内容
- `selection` 中是当前选中的状态
- `redoStack` 和 `undoStack` 就是撤销/重做栈，它是一个数组，存放的是 `ContentState` 类型的编辑器状态
- `decorator` 会寻找特定的模式，并用特定的组件渲染出来


### 什么是 ContentState 

既然编辑器中的内容是存储在一个 `ContentState` 对象中，那么这个 `ContentState` 又是什么？

`ContentState` 也是一个 Immutable Record 对象，其中保存了编辑器里的全部内容和渲染前后的两个选中状态。可以通过 `EditorState.getCurrentContent()` 来获取当前的 `ContentState`，同样调用 `.toJS()` 后将它打印出来看下：
![](/asset/images/2019-07-21-draft-editor-02.png)

`blockMap` 和 `entityMap` 里放置的就是编辑器中的 `block` 和 `entity`，它们是构建 Draft 编辑器的砖瓦。

### 什么是 ContentBlock 和 Entity

一个 `ContentBlock` 表示一个编辑器内容中的一个独立的 block，即视觉上独立的一块。 

以下图的编辑器作为一个例子，图中的四个红框标出的部分都是 block。在平时阅读文章时，内容是以段落为单位的，在编辑器中每个段落就是一个 block，如第一个和最后一个红框中的文字内容。第二个红框中是一张图片，它也是一个 block，但显示方式不同于普通的 block，为了自定义它的显示方式还需要额外做一些工作，后面会加以详细说明。

还有一点需要稍作说明，第三个红框中虽然是空白，但它也是一个 block，只不过其中的文本为空而已。
![](/asset/images/2019-07-21-draft-editor-03.png)

此时，输出一下 `convertToRaw(currentContent)` ，看看其中的内容。注意这里的输出结构与上面的 `currentContent.toJS()` 略有所区别，这里只有 `blocks` 和 `entityMap` 这两项。
![](/asset/images/2019-07-21-draft-editor-04.png)
可以看到 `blocks` 这个数组中依次存放了各个 `block` 的信息，每一个 `block` 都是一个 `contentBlock` 对象。

每个 `contentBlock` 都有如下的几个属性值：
- `key`: 标识出这是哪一个 block
- `type`: 这是何种类型的 block
- `text`: 其中的文字
- ……

Draft.js 中  `block` 的 `type` 有 unstyled，paragraph，header-one，atomic …… 等值，在 Draft.js 的文档中 `atomic` 类型对应的是 `<figure />` 元素，我们也选取了它来实现插入图片的功能。

 图中的这些 block 的除了第三个 key = “1u22q” 的 block 的 type 值是 `atomic` 外，其余的值都是 `“unstyled”`。再仔细看下这个 `atomic` 类型的 block：
![](/asset/images/2019-07-21-draft-editor-05.png)

除了 `key`，`text`，`type` 等值之外，在 `entityRanges` 这一项中保存它保存了使用到的 `entity` 的信息：offset 和 length 确定了 `entity` 在 block 中的范围，而 key 则能让我们去取出对应的 `entity`。

回到上面的打印出的 `contentState`的内容，除了  `blocks` 数组外还有一个 `entityMap`对象。它是以 `entity` 的 `key` 作为键值的对象，里面保存了图片、链接等种类的 `entity` 信息，从中就可获得 `blocks` 所需要的 `entity`。

```js
entityMap: {
	0: { type: "image", mutability: "IMUTABLE", data: {} }
} 
```

以上介绍了 Draft.js 是如何对编辑器中的数据进行存储的，接下来会从代码实现的角度来说明插入图片是如何实现的。


## 插入图片的实现

### 如何插入图片

插入图片有着这样的流程：首先为图片创建一个 `entity`，然后创建一个带有这个 `entity` 的新 `EditorState`，然后更新即可。以下是关键部分的代码：

```js
import { AtomicBlockUtils } from 'draft.js';
// ...

const editorState = this.state.editorState;
const contentState = editorState.getCurrentContent();

// 使用 `contentState.createEntity` 创建一个 `entity`，指定其 `type` 为 `image`
const contentStateWithEntity = contentState.createEntity(
  ‘image’,
  ‘IMMUTABLE’,
  { src }
);

// 获取新创建的 `entity` 的 key
const entityKey = contentStateWithEntity.getLastCreatedEntityKey();


// 用 `EditorState.set()`  来建立一个带有这个 `entity` 的新的 EditorState 
const newEditorState = EditorState.set(
  editorState,
  { currentContent: contentStateWithEntity },
  ‘create-entity’
);

// 利用`AtomicBlockUtils.insertAtomicBlock` 来插入一个新的 `block`
this.setState({
  editorState: AtomicBlockUtils.insertAtomicBlock(newEditorState, entityKey, ‘ ‘)
},
```


### 如何使用 blockRendererFn 来渲染图片

上面我们已经见到了，一张图片是作为一个 `atomic` 类型的 block 插入的。Draft.js 提供了`blockRendererFn` 让我们可以自定义 `ContentBlock` 的渲染方式，给它传入一个函数后，由该函数来判断这个 block 的 `type` 是什么，然后决定如何渲染。

以下的这段代码来自 [Draft.js 的官方文档](https://draftjs.org/docs/advanced-topics-block-components)，展示了如何处理一个  type 为 `atomic` 的 `ContentBlock`。

```js
function myBlockRenderer(contentBlock) {
  const type = contentBlock.getType();
  if (type === 'atomic') {
    return {
      component: MediaComponent,
      editable: false,
      props: {
        foo: 'bar',
      },
    };
  }
}

// Then...
import {Editor} from 'draft-js';
class EditorWithMedia extends React.Component {
  ...
  render() {
    return <Editor ... blockRendererFn={myBlockRenderer} />;
  }
}
```

可以看到这里传递了一个 `props`：

```js
component: MediaComponent,
props: {
	foo: 'bar',
},
```
结果等同于 `<MediaComponent foo='bar' />`，可以利用这里的 `props` 传入所需要的其他数据。

这里我们就可以定义一个自己的 `MediaComponent` 来决定展现方式。因为不管是图片还是视频等其它的媒体类型，它们的 `type` 都是 `atomic`。在 `MediaComponent` 里就需要通过 `entity` 的 `type` 来确定其种类。

```js
const entity = props.contentState.getEntity(props.block.getEntityAt(0));
const { src } = entity.getData();	// 取出图片的地址
const type = entity.getType();  // 判断 entity 的 type 的
```

当 `entity` 的 `type` 是我们自定义的 `image` 时就可以返回 `<Image />` 组件了。

```js
<Image src={src} /> // 自定义的图片组件 <Image />
```

[完整代码可见此文件](https://github.com/noiron/draft-editor/blob/master/src/components/entities/mediaBlockRenderer.js)


### 如何删除一张图片

既然已经插入了图片，那么如何删除它呢？当然我们可以按键盘上的 Backspace 键来删除。也可以在图片的右上角加入一个 “X” 的图标，点击后删除该图片，实现方式如下。


```js
  deleteImage = (block) => {
    const editorState = this.state.editorState;
    const contentState = editorState.getCurrentContent();
    const key = block.getKey();

    const selection = editorState.getSelection();
    const selectionOfAtomicBlock = selection.merge({
      anchorKey: key,
      anchorOffset: 0,
      focusKey: key,
      focusOffset: block.getLength(),
    });

    // 重写 entity 数据，将其从 block 中移除，防止这个 entity 还被其它的 block 引用
    const contentStateWithoutEntity = Modifier.applyEntity(contentState, selectionOfAtomicBlock, null);
    const editorStateWithoutEntity = EditorState.push(editorState, contentStateWithoutEntity, ‘apply-entity’);

    // 移除 block
    const contentStateWithoutBlock = Modifier.removeRange(contentStateWithoutEntity, selectionOfAtomicBlock, ‘backward’);
    const newEditorState =  EditorState.push(editorStateWithoutEntity, contentStateWithoutBlock, ‘remove-range’,);

    this.onChange(newEditorState);
  }
```

至此，对图片的相关操作就完成了。


## 总结与其他

在本文中，介绍了 Draft.js 的基本功能，它是如何进行数据的存储的，及 `EditorState`、`ContentState`、`ContentBlock`、`Entity` 等对象间的关系。并以此为基础说明了如何在编辑器中对图片进行操作。

当然关于 Draft.js 还有很多内容没有在本文中提及，如修改行内文本的样式，利用 `decorators` 来插入与渲染链接等等。这些就需要读者探索下 Draft.js 的官方文档和其他人的分享并亲自尝试下了。


## 参考文章及资源

> [本文所基于的编辑器项目：draft-editor](https://github.com/noiron/draft-editor)

> [How Draft.js Represents Rich Text Data](https://medium.com/@rajaraodv/how-draft-js-represents-rich-text-data-eeabb5f25cf2#.q7vpkxmog)
> [Building a Rich Text Editor with React and Draft.js, Part 2.4: Embedding Images](https://medium.com/@siobhanpmahoney/building-a-rich-text-editor-with-react-and-draft-js-part-2-4-persisting-data-to-server-cd68e81c820)
> [Draft.js 在知乎的实践](https://zhuanlan.zhihu.com/p/24951621)