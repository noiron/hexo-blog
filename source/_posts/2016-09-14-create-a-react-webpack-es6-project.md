---
layout: post
title: 从头创建一个基于 React, webpack, babel 的模板项目 
date:   2016-09-14
tags: [webpack, react.js, es6]
---

如果是一个刚接触 React 的新手，当学完了 React 的各种基本概念和语法之后，准备开始实际的开发工作时，他又会碰到各种新颖的名词：npm, webpack, babel, flux, es2015…… 如果以前接触过这些工具还好，否则为了建立一个简单的项目，还需要学习这一整套的流程，而在这中间又会碰上各种坑，这个过程将会非常痛苦。一个解决方案是去 GitHub 上寻找各种模板项目，用 React + webpack + ... 作为关键字搜索可以发现许多别人创建的空项目，你可以在其基础上稍作修改然后开始开发。我在学习的过程中就是这么做的，而之后对于其中的一些配置项仍是一知半解。

我在这里从头开始创建一个模板项目，并将过程记录下来，一方面希望能给看到这篇文章的人以帮助，另一方面也是加深我自己的理解。

## 项目结构

在开始之前先看看项目的文件结构：

    - src
      - index.js
      - App.js
    - .babelrc
    - index.html
    - package.json
    - server.js
    - webpack.config.js

以上就是我们的项目需要的全部文件了。

接下来我会将这个项目的构建过程分解成几个步骤，你只要按照这些步骤依次进行即可。

<!-- more -->

## 初始化项目

首先我们新建一个名为 react-webpack-boilerplate 的文件夹，然后需要在命令行下对项目进行初始化。

假设你已经对 NPM 有了基本的了解，在命令行下进入项目目录后，运行：

    npm init

之后你需要按照提示输入你的项目的基本信息，这里也可以一直回车，则选择默认选项，或者运行：

    npm init -y    // 所有信息都按照默认设置

`npm init` 命令的作用就是初始化一个项目，你可以发现项目文件夹内多出了一个 `package.json` 文件，它的内容如下：

```javascript
    // 按照默认设置生成的 package.json 文件
    {
      "name": "react-webpack-boilerplate",
      "version": "1.0.0",
      "description": "",
      "main": "index.js",
      "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1"
      },
      "keywords": [],
      "author": "",
      "license": "ISC"
    }
```

项目的运行需要依赖各种 npm 包，需要在文件的 "dependencies" 或 "devDependencies" 项下写明 npm 包的名称及版本。现在这个项目还是空的，所以这里还没有这两项。

由于直接用 npm 安装的速度很慢，我们可以考虑用 cnpm install 命令来代替 npm install 命令。

CNPM 是淘宝建立的一个 NPM 镜像，由于服务器在国内，速度自然也就快多了。

    // 安装 cnpm
    $ npm install -g cnpm --registry=https://registry.npm.taobao.org

完成后，你就可以用 cnpm install 来安装所需的 npm 包了。

> [淘宝 NPM 镜像](https://npm.taobao.org/)


## 安装及配置 webpack

### webpack 是什么？

webpack 是一个打包工具。

### webpack 的安装

继续在命令行下面运行：

    npm install webpack --save-dev

这一句的作用的是给项目安装 webpack，现在查看项目文件夹，会发现多出了一个 `node_modules` 文件夹，这里就是存放项目所依赖的 npm 包的地方。

上面的这条命令后面跟着的 `--save-dev`，表示将会把 webpack 的名称写入 `package.json` 文件中。我们打开 `package.json` 文件，发现文件中多了一项：

    "devDependencies": {
        "webpack": "^1.13.2"
    }

`--save-dev` 表示会写入 "devDependencies" 项中，而 `--save` 写入的是 "dependencies" 项。

### webpack 的测试

你可以选择将 webpack 安装在全局中，或者安装在这一个项目中。（全局安装和本地安装的区别）

选择全局安装：

    $ cnpm install webpack -g 

既然我们已经将 webpack 安装在了全局，那就先在命令行下测试一下它的打包功能。

新建 `index.js` 文件，里面只有一行：

```javascript
    console.log('This is a javascript file for test.');
```

运行一个最简单的 webpack 命令：

    $ webpack index.js bundle.js

它的意思是将 `index.js` 文件打包成名为 `bundle.js` 的文件，后一个参数是你想要生成的文件名称。

运行之后会发现多出了一个 `bundle.js` 文件，用 node 运行该文件测试一下：

    $ node bundle.js
    This is a javascript file for test.    // 输出了正确的结果

可以新建一个 html 文件，引入 `bundle.js`:

```html
    // index.html
    <!DOCTYPE html>
    <html>
      <head>
        <title>react webpack boilerplate</title>
      </head>
      <body>
        <script src="./bundle.js"></script>
      </body>
    </html>
```

打开 `index.html`，你可以在浏览器的控制台中看到输出结果。

### webpack 的配置文件

我们可以在命令行下完成各种复杂的 webpack 操作，不过更方便的方法是利用配置文件。(命令行下的参数有哪些？)

新建 `webpack.config.js` 文件：

```javascript
    // 一个简单的 webpack.config.js 文件
    var path = require('path');
    var webpack = require('webpack');

    module.exports = {
      entry: [
        './index'
      ],
      output: {
        path: path.join(__dirname, 'dist'),
        filename: 'bundle.js',
      }
    }
```
依次解释 webpack.config.js 中的各项。

新建 a.js，用 require 语法。

html 文件的引用路径改成：

```html
    <script src="./dist/bundle.js"></script>
```

以上的步骤可以在 step1 文件夹内查看。

### 安装 web-dev-server，并用其开启一个服务器

首先运行如下命令，安装 web-dev-server：

    $ npm i webpack-dev-server --save-dev

添加如下的 server.js 文件：

```javascript
    // server.js
    var webpack = require('webpack');
    var WebpackDevServer = require('webpack-dev-server');
    var config = require('./webpack.config.js');

    var server = new WebpackDevServer(webpack(config));

    server.listen(3000, 'localhost', function(err, result) {
      if (err) {
        return console.log(err);
      }
      return console.log('listening at locahost:3000...');
    })

    var server = new WebpackDevServer(webpack(config));
```

这一句创建了一个 webpack dev server，这是一个 node.js express 服务器，关于它的用法可以见[这里的文档](https://webpack.github.io/docs/webpack-dev-server.html)。

好了，现在你可以运行：

    $ node server.js

然后打开浏览器进入 http://localhost:3000 查看效果了（当然，现在除了在控制台输出了信息之外什么都没干）

在 `package.json` 文件内加上这么一项：

```javascript
  "scripts": {
    "start": "node server.js"
  },
```

你可以用 `npm start` 来代替上面的 `node server.js` 了，这两者是等价的。


上述步骤可以到 step2 文件夹中查看。


## 有关 Babel 的一切

### Babel 是什么？

因为在 React 的项目中会用到 ES6(即ES2015) 的语法，而 ES6 还没有得到广泛的支持，所以我们需要借用 Babel 这个工具将我们使用了 ES6 语法的代码转换成 ES5 的语法，从而可以在更广泛的环境下运行。

> [Babel 的官方网站](https://babeljs.io/)


### 怎么安装 Babel？

在我们的项目中，需要安装这样的几个 package：

    $ npm install --save-dev babel-loader babel-core

在 Babel 6.x 之前的版本中，Babel 会执行一些默认的转换规则，而在 6.x 之后的版本，你需要显示地告诉 babel 你想执行什么转换。最简单的方法就是使用 preset，比如说 ES2015 Preset。

    $ npm install babel-preset-es2015 --save-dev

> Presets are sharable .babelrc configs or simply an array of babel plugins.

我们这里还需要另一个名为 stage-0 的 Preset：

    $ npm install babel-preset-es2015 --save-dev

> 参考文档：
> [Babel 的安装](https://babeljs.io/docs/setup/#installation)
> [Presets](https://babeljs.io/docs/plugins/#presets)

现在只要添加一个 .babelrc 文件，在里面写上如下内容即可：

    {
      "presets": ["es2015", "stage-0"]
    }

所以 .babelrc 文件是什么？





将 webpack.config.js 文件做如下的修改：

```javascript
    module: {
      loaders: [{
        test: /\.js$/,
        loaders: ['babel'],
        include: path.join(__dirname, 'src')
      }]
    },
    devServer: {
      stats: 'errors-only'
    }
```

output 这一项中加上：

    publicPath: '/static/'

server.js 修改：

    var server = new WebpackDevServer(webpack(config), {
      stats: config.devServer.stats,
        publicPath: config.output.publicPath
    });


这样就可以在我们的项目使用 babel 来进行转码了。

这一步可在 step 3 中查看。

## 安装 React

既然你在看这篇文章，相信你已经对 React 有了一定的了解，React 是做什么的就不用再多介绍了。

安装 React 的 package 十分简单：

npm install react react-dom --save

为了能够让 babel 对 react 进行处理，再安装一个 babel-preset-react

npm install babel-preset-react

同时在 .babelrc 文件中也加入这一项：

  "presets": ["es2015", "stage-0", "react"]


新建一个基本的 React 组件试试看：

首先在 index.html 中加上这样的一个元素：

    <div id="app"></div>

React 元素将在上面渲染。

在 src 文件夹下新建两个文件 App.js index.js

    ./src/index.js
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App.js';

ReactDOM.render(
  <App />,
  document.getElementById('app')
);


./src/App.js
import React, { Component } from 'react';

export default class App extends Component {
  render() {
    console.log('%c%s', 'font-size:20px;color:red', 'Something happened.');

    return (
      <div>This is a react boilerplate project with webpack and es6.</div>
    );
  };
}


现在你可以 npm start 看看效果了。

以上内容可以在 step4 文件夹中查看。

## 安装 react-hot-loader

其实到上一步为止，我们已经完成了一个完整的 react 模板项目，不过还缺一点，就是当我们在编辑器中修改了文件的时候，需要在浏览器里手动刷新，才能看到结果。我们可以利用一个名为 react-hot-loader 的工具来帮助我们实现浏览器自动刷新的效果。

首先安装 react-hot-loader:

    npm install react-hot-loader@^1.3.0 --save-dev

这里加上了版本号是因为默认安装最新的 react-hot-loader 为（），设置会和下面的有所区别。

在 server.js 中加一行：

var server = new WebpackDevServer(webpack(config), {
  stats: config.devServer.stats,
  hot: true,
  publicPath: config.output.publicPath
});

对 webpack.config.js 做如下修改：

  entry: [
    'webpack-dev-server/client?http://localhost:3000',
    'webpack/hot/only-dev-server',
    './src/index'
  ],

    loaders: [{
      test: /\.js$/,
      loaders: ['react-hot', 'babel'],
      include: path.join(__dirname, 'src')
    }]

加入一个插件：

  plugins: [
    new webpack.HotModuleReplacementPlugin()
  ]

这一步的内容可以查看 step5 文件夹。



到此我们的这个模板项目就完成了。你可以在GitHub上看到完整的项目代码。