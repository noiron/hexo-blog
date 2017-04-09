---
layout: post
title: 课程笔记--Building a javascript development environment 
date:   2017-03-05
tags: [webpack, javascript, npm]
---

这篇博客是我在学习 [Pluralsight上的课程：Building a JavaScript Development Environment](https://app.pluralsight.com/library/courses/javascript-development-environment/recommended-courses) 时记下的笔记，仅作个人记录之用。

这门课程系统地介绍了前端开发中的许多工具和术语，从编辑器的选择到项目的部署都进行了介绍，我在学习了这个课程后感觉将之前零散学习的内容串了起来。

顺便一说，Pluralsight网站上面有许多不错的课程，不过价格太贵，需每月付29美元😥。但是，你可以用微软账号加入[Visual Studio Dev Essentials 计划](https://www.visualstudio.com/zh-hans/dev-essentials/)，这样就可以免费获得Pluralsight 3个月的订阅了😀。

---

## 1. Course Overview

编程的时候要做许多选择

- Editor
- Module format
- HTML generation
- Transpiling
- Bundler
- Linting
- Testing
- ...

这门课程介绍了如何在这些选项中做出选择

<!-- more -->

## 2. You need a starter kit

### You need a starter kit

原因：

- Codifies
    - 决定
    - 最佳实践
    - 教训
- 鼓励一致性
- 避免忘记重要的细节
- 增加质量
    - Doing the "right" thing is the easy thing
- 避免重复工作

### A starter kit is an automated checklist

### Who is this course for?

Atwood's Law 

JavaScript is Everywhere

### What Belongs in your starter kit?

- Package Management
- Bundling
- Minification
- Sourcemaps
- Transpiling
- Dynamic HTML Generation
- Centralized HTTP
- Mock API framework
- Component libraries
- Development Webserver
- Linting
- Automated Testing
- Continuous Integration
- Automated deployment
- Working example app


### Agenda

Rhythm
1. Options
2. Recommendation
3. Implement

---

## 3. Editors and Configuration

### What to look for in a JavaScript Editor

- Strong ES2015+ support
- framework intelligence
- Built-in terminal

可选编辑器：
- Atom
- WebStorm
- Brackets
- VSCode

### Editorconfig

EditorConfig 保证大家的编辑器设置一样。

### Demo: Editorconfig

增加 .editorconfig 文件

---

## 4. Package Management

### Package Managers

- Bower
- npm
- JSPM
- ...

### Demo: Install Node and npm Packages

### Package Security

何时 Run Security Check ?

npm start 

### Demo: Node Security Platform

全局安装 nsp

    nsp install -g nsp

    λ nsp check
    (+) No known vulnerabilities found

---

## 5. Development Web Server

### Development Web Servers

- http-server
- live-server
- Express
- budo
- Webpack dev server
- Browsersync

### Sharing Work-in-progress

#### localtunnel

步骤：

    1. npm install localtunnel -g
    2. start app
    3. lt --port 3000


#### ngrok

1. 注册
2. 安装 ngrok
3. 安装 authtoken
4. start app
5. ./ngrok http 80

#### now

#### surge

快速将静态文件部署到公开URL上去。

1. npm install -g surge
2. surge



## 6. Automation

### Automation Options

- Grunt
- Gulp
- npm scripts

### Demo: Pre/Post Hooks

```json
scripts: {
    "prestart": "node buildScripts/startMessage.js",
    "start": "node buildScripts/srcServer.js",
    "poststart": "..."
}
```

当运行 `npm start` 时，`prestart`命令中的内容会自动在其之前运行。

### Demo: Create Security Check and Share Scripts

在 scripts 项中加入：

    "security-check": "nsp check"

### Demo: Concurrent Tasks

假设在运行server的同时，进行security-check，对start命令进行如下的修改：

    "start": "npm-run-all --parallel security-check open:src",
    "open:src": "node buildScripts/srcServer.js",

其中用 npm-run-all 来代替。

可以用 npm start -s 减少输出中的噪声。



## 7. Transpiling

### JavaScript Versions

### Transpilers

Popular transpilers:

- Babel
- TypeScript
- Elm

Babel可以将最新版本的JavaScript代码向下转换成ES5，从而可以使用所有的新特性。也是最流行的。

TypeScript是JavaScript的超集。 

Elm不是JavaScript，可以编译成JS。

Babel配置风格：

1. 使用.babelrc文件：不止能用在npm中，独立文件易于阅读

2. package.json：项目中少了一个文件

在文件中加入`babel`项即可

```json
{
    "name": "my-package",
    "version": "1.0.0",
    "babel": {
        // my babel config here
    }
}
```

### Transpiling Build Scripts

### Demo: Set Up Babel

加入 .babelrc文件

```
{
  "presets": [
    "latest"
  ]
}
```

在preset命令中用babel-node来代替node，来使用es6语法。这样可以使用import, const 等关键字。

```json
"prestart": "babel-node buildScripts/startMessage.js",
```

---

## 8. Bundling

### Intro

CommonJS不能在web浏览器中工作，需要将npm package打包成浏览器能够使用的形式。

### Module Formats

5中模块形式：
- IIFE
- AMD
- CommonJS
- UMD
- ES6 Modules

现在应继续使用的两种模块：

- CommonJS: node中使用

    var jquery = require('jquery')

- ES6 Module

### Why ES6 Modules?

原因：

- Standardized
- Statically analyzable
    - Improved autocomplete
    - intelligent refactoring
    - Fails fast
    - Tree shaking
- Easy to read
    - Named imports
    - Default exports


### Choosing a Bundler

- require.js: first popular bundler
- Browserify: large plugin ecosystem
- Webpack: bundles more than just JS
- Rollup: tree shaking, fast loading production code, quite new ...
- JSPM: load modules at runtime

Why webpack?
- Much more than just js
    - css
    - images
    - fonts
    - html
- Bundle splitting
- Hot module reloading
- Webpack2 offers tree shaking

### Demo: Configuring Webpack

在根目录下新建`webpack.config.dev.js`文件，内容如下：

```javascript
import path from 'path';

export default {
  debug: true,
  devtool: 'inline-source-map',
  noInfo: false,
  entry: [
    path.resolve(__dirname, 'src/index')
  ],
  target: 'web',
  output: {
    path: path.resolve(__dirname, 'src'),
    publicPath: '/',
    filename: 'bundle.js'
  },
  plugins: [],
  module: {
    loaders: [
      { test: /.js$/, exclude: /node_modules/, loaders: ['babel']},
      { test: /\.css$/, loaders: ['style', 'css']}
    ]
  }
}
```

Note: Webpack won't actually generate any physical files for our development build. It will serve our build from memory.


### Demo: Configure Webpack with Express

在 `buildScripts/srcServer.js`中添加下列内容

```javascript
import webpack from 'webpack';
import config from '../webpack.config.dev';

// ...

const compiler = webpack(config);

app.use(require('webpack-dev-middleware')(compiler, {
  noInfo: true,
  publicPath: config.output.publicPath
}));
```

### Demo: Create app entry point

添加`index.js`文件

```javascript
// src/index.js
import numeral from 'numeral';

const courseValue = numeral(1000).format('$0, 0.00');
console.log(`I would pay ${courseValue} for this awesome course!`);
```

```html
    <script src="bundle.js"></script>
```

### Demo: Handling css with webpack

在我们的webpack配置文件中已经加入了css的loader的情况下，只需要在index.js引入css文件即可。

```javascript
import './index.css';
```

### Sourcemaps

Why: Debug transpiled and bundled code.

Maps code back to original source


### Demo: Debugging via sourcemaps



## 9. Linting

### Why Lint?

- Enforce consistency
- Avoid mistakes

### Linters

Linter选择:

- JSLint
- JSHint
- **ESLint**

### ESLint Configuration Decisions Overview

1. Config format?
2. Which built-in rules?
3. Warnings or errors?
4. Which plugins?
5. Use preset instead?

### Decision 1: Configuration File Fomat

创建ESLint配置文件，或在`package.json`文件中加入`eslintConfig`项。

### Decision 2: Which rules

### Decision 3: Warnings or errors

- Warnings
    - Can continnue development
    - Can be ignored

- Error
    - Breaks the build
    - Cannot be ignored

### Decision 4: Which plugins

React开发推荐：eslint-plugin-react

[Awesome ESLint](https://github.com/dustinspecker/awesome-eslint)

### Decision 5: Preset

- From scratch
- Recommended
- Presets

### Watching Files with ESLint

Issue: ESLint doesn't watch files

解决方案：

- eslint-loader
    - Re-lint all files upon save
- eslint-watch
    - ESLint wrapper that adds file watch
    - Not tied to webpack
    - Better warning/error formatting
    - ...

### Linting Experimental Features

- Run ESLint directly
    - Supports ES6 and ES7 natively
- Babel-eslint
    - Also lints stage 0-4 features

We'll use plain ESLint and run it via eslint-watch.

### Why Lint Via an antomated Build?

1. One place to check
2. Universal Configuration
3. Part of continuous integration

### Demo: ESLint Set Up

ESLint设置：

- ESLint Recommended
- eslint-watch

添加`.eslintrc.json`文件，内容如下：

[GitHubGist](https://gist.github.com/coryhouse/61f866c7174220777899bcfff03dab7f)

创建调用`eslint-watch`的 npm script：

    "lint": "esw webpack.config.* src buildScripts --color",

可以使用注释来排除某一个eslint的规则：

    /* eslint-disable no-console */

    // eslint-disable-line no-console

### Demo: Watching Files

在 lint script 下加一句：

    "lint:watch": "npm run lint -- --watch",

加入 start script 中：

    "start": "npm-run-all --parallel security-check open:src lint:watch",



## 10. Testing and Continuous Integration

### Test Decisions Overview

| Style | Focus |
| ----- | ----- |
| Unit | Single function or module |
| Integration | Interations between modules |
| UI | Automate interaations with UI |

### Decision 1: Testing Framework

- **Mocha**
- Jasmine
- Tape
- QUnit
- AVA
- Jest

以上任一皆可。

### Decision 2: Assertion Libries

Jasmine and Jest have assertion libries built in.

Assertion: Declare what you expect
    
    eg. expect(2 + 2).to.equal(4)

- **Chai**
- Should.js
- expect

### Decision 3: Helper Libraries

- JSOM: Simulate the browser's DOM

- Cheerio: jQuery for the server

### Decision 4: Where to run tests

- Browser: Karma, Testem
- Headless Browser: PhantomJS
- In-mermory DOM: **JSDOM**

### Decision 5: Where do test files belong

- Centralized: in a folder called 'test'
    - Less "noise" in src folder
    - Deployment confusion
    - Inertia
- Alongside
    - Easy imports
    - Clear visibility
    - Conventient to open
    - ...

### Decision 6: When should tests run?

- Unit tests should run when you hit save
    - Rapid feedback
- Interation tests often run on demand, or in QA

### Demo: Testing Setup

```javascript
/** buildScripts/testSetup.js */
// Register babel to transpile before our test run.
require('babel-register')();

// Disable webpack features that Mocha doesn't understand.
require.extensions['.css'] = function(){};
```


```json
// add "test" scripts in package.json
"test": "mocha --reporter progress buildScripts/testSetup.js \"src/**/*.test.js\" "
```

```javascript
/* src/index.test.js */
import { expect } from 'chai';

describe('Our first test', () => {
  it('should pass', () => {
    expect(true).to.equal(true);
  })
})
```

### Demo: DOM Testing

```javascript
import jsdom from 'jsdom';
import fs from 'fs';

// other code

describe('index.html', () => {
  it('should say hello', (done) => {
    const index = fs.readFileSync('./src/index.html', "utf-8");
    jsdom.env(index, function(err, window) {
      const h1 = window.document.getElementsByTagName('h1')[0];
      expect(h1.innerHTML).to.equal("Hello World!");
      done();
      window.close();
    })
  })
});
```

### Demo: Watching Tests

Add this script to `package.json`:

    "test:watch": "npm run test -- --watch"

Then add `test:watch` to the end of `start` script:

    "start": "npm-run-all --parallel security-check open:src lint:watch test:watch"

### Why Continuous Integration?

When team commits code, confirm immediately that the commit works as expected when on another machine.

### What Does Continuous Integration Do?

- Run automated build
- Run your tests
- Check code coverage
- Automate deployment

### Choosing a CI Server

- **Travis**: Linux based
- **Appveyor**: Windows based
- Jenkins
- CircleCI
- Semaphore
- SnapCI

### Demo: Travis CI

使用GitHub账号登录[Travis CI](https://travis-ci.org/)，即可选择你在GitHub上的项目进行持续集成。

需要在项目中加入travis的配置文件`.travis.yml`：

    language: node_js
    node_js:
      - "6"

### Demo: Appveyor



## 11. HTTP Calls

### HTTP Call Approaches

- Node
    - http
    - request
- Browser
    - XMLHttpRequest
    - jQuery
    - Framework-based
    - Fetch
- Node & Browser
    - isomorphic-fetch
    - xhr
    - superagent
    - axios

### Centralizing HTTP Requests

Why?

- Configure all calls
- Handle preloader logic
- Handle errors
- Single seam for mocking

### Demo: Fetch

修改 `srcServer.js` 文件

```javascript
app.get('/users', function(req, res) {
  res.json([
    {"id": 1,"firstName":"Bob","lastName":"Smith","email":"bob@gmail.com"},
    {"id": 2,"firstName":"Tammy","lastName":"Norton","email":"tnorton@yahoo.com"},
    {"id": 3,"firstName":"Tina","lastName":"Lee","email":"lee.tina@hotmail.com"}
  ]);
});
```

加入 userApi.js 文件

```javascript
import 'whatwg-fetch';

export function getUsers() {
  return get('users');
}

function get(url) {
  return fetch(url).then(onSuccess, onError);
}

function onSuccess(response) {
  return response.json();
}

function onError(error) {
  console.log(error); // eslint-disable-line no-console
}
```

此时打开`localhost:3000/users`，能查看API结果。


### Selective Polyfilling

Polyfill.io: Only send polyfill to those who need it

### Why Mock HTTP?

- Unit Testing
- Instant response
- Keep working when services are down
- Rapid prototyping
- Avoid inter-team bottlenecks
- Work offline

### How to Mock HTTP

- Nock
- Static JSON
- Create development webserver
    - api-mock
    - JSON server
    - JSON Schema faker
    - Browsersync
    - Express, etc

### Our Plan for Mocking

1. Declare our schema:
    - JSON Schema Faker
2. Generate Random Data:
    - faker.js
    - chance.js
    - randexp.js
3. Serve Data via API
    - JSON Server


### Mocking Libraries

JSON Schema Faker

JSON Server

### Demo: Creating a Mock API Data Schema

- Mock HTTP

创建文件：`buildScripts/mockDataSchema.js`文件，内容如下：bit.ly/ps-mock-data-schema


### Demo: Generating Mock Data

在`buildScripts`中新建文件`generateMockData.js`:

```javascript
// buildScripts/generateMockData.js
import jsf from 'json-schema-faker';
import { schema } from './mockDataSchema';
import fs from 'fs';
import chalk from 'chalk';

const json = JSON.stringify(jsf(schema));

fs.writeFile("./src/api/db.json", json, function (err) {
  if (err) {
    return console.log(chalk.red(err));
  } else {
    console.log(chalk.green("Mock data generated"));
  }
})
```

在`package.json`中加入`generate-mock-data` script:

    "generate-mock-data": "babel-node buildScripts/generateMockData"

运行`npm run generate-mock-data`，会生成一个`db.json`文件。

### Demo: Serving Mock Data via JSON Server

Add script:

    "start-mockapi": "json-server --watch src/api/db.json --port 3001"

json-server 会将`db.json`中的内容生成一个API，如下为运行后输出内容。

    Loading src/api/db.json
    Done

    Resources
    http://localhost:3001/users

    Home
    http://localhost:3001

在开发环境中需要将API换成 mock api。

```javascript
// src/api
export default function getBaseUrl() {
  const inDevelopment = window.location.hostname === 'localhost';
  return inDevelopment ? 'http://localhost:3001/' : '/';
}
```

```javascript
function get(url) {
  // return fetch(url).then(onSuccess, onError);
  return fetch(baseUrl + url).then(onSuccess, onError);
}
```

### Demo: Manipulating Data via JSON Server

```javascript
// src/api
export function deleteUser(id) {
  return del(`users/${id}`);
}

function del(url) {
  const request = new Request(baseUrl + url, {
    method: 'DELETE'
  });

  return fetch(request).then(onSuccess, onError);
}
```

---

## 12. Project Structure

### Why a Demo App?

Examples of:
- Directory structure and file naming
- Framework usage
- Testing
- Mock API
- Automated deployment

### Tip 1: JS Belongs in a .js File

```javascript
<script>
    // dont't put code here
</script>
```

### Tip 2: Consider Organizing by Feature

### Tip 3: Extract Logic to POJOs

POJOs: Plain Old JavaScript Objects, Pure logic, No framework-specific code



## 13. Production Build

### Minification and Sourcemaps

### Demo: Production Webpack Configuration with Minification

新建`webpack.config.prod.js`，相比`webpack.config.dev.js`加入plugin：

```javascript
  plugins: [
    // Eliminate duplicate packages when generating bundle
    new webpack.optimize.DedupePlugin(),

    // Minify JS
    new webpack.optimize.UglifyJsPlugin()
  ],
```

新建`buildScripts/build.js`文件。

### Demo: Configure Local/dist Server

新建`buildScripts/distServer.js`文件，用于 static files。

### Demo: Toggle Mock API

### Demo: Production Build npm Scripts

### Dynamic HTML Generation

### Bundle Splitting

### Cache Busting

### Demo: Extract and Minify CSS

### Error Logging


## 14. Production Deploy