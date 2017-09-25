---
layout: post
title: React Router v4 之代码分割：从放弃到入门
date: 2017-09-25
tags: [react-router, webpack]
---

## 背景介绍

React Router v4 推出已有六个月了，网络上因版本升级带来的哀嚎仿佛就在半年前。我在使用这个版本的 React Router 时，也遇到了一些问题，比如这里所说的代码分割，所以写了这篇博客作为总结，希望能对他人有所帮助。

## 什么是代码分割（code splitting）

在用户浏览我们的网站时，一种方案是一次性地将所有的 JavaScript 代码都下载下来，可想而知，代码体积会很可观，同时这些代码中的一部分可能是用户此时并不需要的。另一种方案是按需加载，将 JavaScript 代码分成多个块（chunk），用户只需下载当前浏览所需的代码即可，用户进入到其它页面或需要渲染其它部分时，才加载更多的代码。这后一种方案中用到的就是所谓的**代码分割（code splitting）**了。

<!-- more -->

当然为了实现代码分割，仍然需要和 webpack 搭配使用，先来看看 webpack 的文档中是如何介绍的。

[Webpack 文档的 code splitting 页面](https://webpack.js.org/guides/code-splitting/)中介绍了三种方法：

1. 利用 webpack 中的 `entry` 配置项来进行手动分割
2. 利用 `CommonsChunkPlugin` 插件来提取重复 chunk
3. 动态引入（Dynamic Imports）

你可以读一下此篇文档，从而对 webpack 是如何进行代码分割的有个基本的认识。本文后面将提到的方案就是基于上述的第三种方法。

## React Router 中如何进行代码分割

在 v4 之前的版本中，一般是利用 `require.ensure()` 来实现代码分割的，而在 v4 中又是如何处理的呢？

### 使用 bundle-loader 的方案

在 React Router v4 官方给出的[文档](https://reacttraining.com/react-router/web/guides/code-splitting)中，使用了名为 [`bundle-loader`](https://github.com/webpack-contrib/bundle-loader) 的工具来实现这一功能。

其主要实现思路为创建一个名为 `<Bundle>` 的组件，当应用匹配到了对应的路径时，该组件会动态地引入所需模块并将自身渲染出来。

示例代码如下：

```js
import loadSomething from 'bundle-loader?lazy!./Something'

<Bundle load={loadSomething}>
  {(mod) => (
    // do something w/ the module
  )}
</Bundle>
```

更多关于 `<Bundle>` 组件的实现可参见上面给出的文档地址。

### 使用 bundle-loader 方法存在的不足之处

这里提到的两个缺点我们在实际开发工作中遇到的，与我们的项目特定结构相关，所以你可能并不会遇上。

一、 代码丑陋

由于我们的项目是从 React Router v2, v3 升级过来的，在之前的版本中对于异步加载的实现采用了集中配置的方案，即项目中存在一个 `Routes.js` 文件，整个项目的路径设置都放在了该文件中，这样方便集中管理。

但是在 React Router v4 版本中，由于使用了 `bundle-loader` 来实现代码分割，必须使用以下写法来引入组件：

```js
import loadSomething from 'bundle-loader?lazy!./Something'
```

而我们的 `reducer` 和 `saga` 文件也需要使用此种方法引入，导致 `Routes.js` 文件顶端将会出现一长串及其冗长的组件引入代码，不易维护。如下所示：

![](/asset/images/2017-08-14-bundle-loader.png)

当用这种方法引入的模块数量过多时，文件将会不忍直视。

二、 存在莫名的组件生命周期Bug

在使用了这种方案后，在某些页面中会出现这样的一个Bug：应用中进行页面跳转时，上一个页面的组件会在 `unmount` 之后重新创建一次。表现为已经到了下一页面，但是会调用存在于跳转前页面中的组件的 `componentDidMount` 方法。

当然，这个Bug只与我自己的特定项目有关，错误原因可能与 `bundle-loader` 并无太大关联。不过因为一直无法解决这一问题，所以决定换一个方案来代替 `bundle-loader`。

## 使用 import() 的新方案

Dan Abramov 在这个 create-react-app 的 issue 中给出了 `bundle-loader` 的替代方案的链接：[Code Splitting in Create React App](http://serverless-stack.com/chapters/code-splitting-in-create-react-app.html)，可以参考该篇文章来实现我们项目中的代码分割功能。

### 代码分割和 React Router v4

一个常规的 React Router 项目结构如下：

```js
// 代码出处：
// http://serverless-stack.com/chapters/code-splitting-in-create-react-app.html

/* Import the components */
import Home from './containers/Home';
import Posts from './containers/Posts';
import NotFound from './containers/NotFound';

/* Use components to define routes */
export default () => (
  <Switch>
    <Route path="/" exact component={Home} />
    <Route path="/posts/:id" exact component={Posts} />
    <Route component={NotFound} />
  </Switch>
);
```

首先根据我们的 route 引入相应的组件，然后将其用于定义相应的 `<Route>`。

但是，不管匹配到了哪一个 route，我们这里都一次性地引入所有的组件。而我们想要的效果是当匹配了一个 route，则只引入与其对应的组件，这就需要实现代码分割了。

### 创建一个异步组件（Async Component）

异步组件，即只有在需要的时候才会引入。

```js
import React, { Component } from 'react';

export default function asyncComponent(importComponent) {

  class AsyncComponent extends Component {

    constructor(props) {
      super(props);

      this.state = {
        component: null,
      };
    }

    async componentDidMount() {
      const { default: component } = await importComponent();

      this.setState({
        component: component
      });
    }

    render() {
      const C = this.state.component;

      return C
        ? <C {...this.props} />
        : null;
    }

  }

  return AsyncComponent;
}
```

`asyncComponent` 接收一个 `importComponent` 函数作为参数，`importComponent()` 在被调用时会动态引入给定的组件。

在 `componentDidMount()`中，调用传入的 `importComponent()`，并将动态引入的组件保存在 state 中。

### 使用异步组件（Async Component）

不再使用如下静态引入组件的方法：

```js
import Home from './containers/Home';
```

而是使用 `asyncComponent` 方法来动态引入组件：

```js
const AsyncHome = asyncComponent(() => import('./containers/Home'));
```

此处的 `import()` 来自于新的 ES 提案，其结果是一个 Promise，这是一种动态引入模块的方法，即上文 webpack 文档中提到的第三种方法。更多关于 `import()` 的信息可以查看这篇文章：[ES proposal: import() – dynamically importing ES modules](http://2ality.com/2017/01/import-operator.html)。

注意这里并没有进行组件的引入，而是传给了 `asyncComponent` 一个函数，它将在 `AsyncHome` 组件被创建时进行动态引入。同时，这种传入一个函数作为参数，而非直接传入一个字符串的写法能够让 webpack 意识到此处需要进行代码分割。

最后如下使用这个 `AsyncHome` 组件：

```js
<Route path="/" exact component={AsyncHome} />
```

### 对于 reducer 和 saga 文件的异步加载

在上面的这篇文章中，只给出了对于组件的异步引入的解决方案，而在我们的项目中还存在将 `reducer` 和 `saga` 文件异步引入的需求。

```js
processReducer(reducer) {
    if (Array.isArray(reducer)) {
        return Promise.all(reducer.map(r => this.processReducer(r)));
    } else if (typeof reducer === 'object') {
        const key = Object.keys(reducer)[0];
        return reducer[key]().then(x => {
            injectAsyncReducer(key, x.default);
        });
    }
}
```

将需要异步引入的 reducer 作为参数传入，利用 Promise 来对其进行异步处理。在 `componentDidMount` 方法中等待 reducer 处理完毕后在将组件保存在 state 中，对于 saga 文件同理。

```js
// componentDidMount 中做如下修改
async componentDidMount() {
    const { default: component } = await importComponent();

    Promise.all([this.processReducer(reducers),
            this.processSaga(sagas)]).then(() => {
        this.setState({
            component
        });
    });
}
```

在上面对 `reducer` 文件进行处理时，使用了这样的一行代码：

```js
injectAsyncReducer(key, x.default);
```

其作用是利用 Redux 中的 `replaceReducer()` 方法来修改 reducer，具体代码见下。

```js
// reducerList 是你当前的 reducer 列表
function createReducer(asyncReducers) {
    asyncReducers
    && !reducersList[Object.keys(asyncReducers)[0]]
    && (reducersList = Object.assign({}, reducersList, asyncReducers));
    return combineReducers(reducersList);
}

function injectAsyncReducer(name, asyncReducer) {
    store.replaceReducer(createReducer({ [name]: asyncReducer }));
}
```

完整的 asyncComponent 代码可见以下 gist，注意一点，为了能够灵活地使用不同的 `injectAsyncReducer`, `injectAsyncSaga` 函数，代码中使用了高阶组件的写法，你可以直接使用内层的 `asyncComponent` 函数。

<script src="https://gist.github.com/noiron/7f774dea55bcc52921e5bc5a50c2aa10.js"></script>

## asyncComponent 方法与 React Router v4 的结合使用

### 组件、reducer、saga 的异步引入

考虑到代码可读性，可在你的 `route` 目录下新建一个 `asyncImport.js` 文件，将需要异步引入的模块写在该文件中：

```js
// 引入前面所写的异步加载函数
import asyncComponent from 'route/AsyncComponent';

// 只传入第一个参数，只需要组件
export const AsyncHomePage = asyncComponent(() => import('./homepage/Homepage'));

// 传入三个参数，分别为 component, reducer, saga
// 注意这里的第二个参数 reducer 是一个对象，其键值对应于redux store中存放的键值
export const AsyncArticle = asyncComponent(
    () => import('./market/Common/js/Container'),
    { market: () => import('./market/Common/js/reducer') },
    () => import('./market/Saga/watcher')
);

// reducer 和 saga 参数可以传入数组
// 当只有 saga，而无 reducer 参数时，第二项参数传入空数组 []
const UserContainer = () => import('./user/Common/js/Container');
const userReducer = { userInfo: () => import('./user/Common/js/userInfoReducer') };
const userSaga = () => import('./user/Saga/watcher');
export const AsyncUserContainer = asyncComponent(
    UserContainer,
    [userReducer, createReducer],
    [userSaga, createSaga]
);
```

然后在项目的 `Router` 组件中引用：

```js
// route/index.jsx
<Route path="/user" component={AsyncArticle} />
<Route path="/user/:userId" component={AsyncUserContainer} />
```

根据 React Router v4 的哲学，React Router 中的一切皆为组件，所以不必在一个单独的文件中统一配置所有的路由信息。建议在你最外层的容器组件，比如我的 `route/index.jsx` 文件中只写入**对应一个单独页面的容器组件**，而页面中的子组件在该容器组件中异步引入并使用。

<del>在 React Router v5 发布之前完成了本文，可喜可贺👏</del>

## 参考资料

> [Code Splitting in Create React App](http://serverless-stack.com/chapters/code-splitting-in-create-react-app.html)
> [ES proposal: import() – dynamically importing ES modules](http://2ality.com/2017/01/import-operator.html)
