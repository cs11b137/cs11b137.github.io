+++
title = "从零搭一个前端构建脚手架：Webpack 实践"
date = 2019-09-05
draft = false
categories = ["前端工程化"]
tags = ["Webpack", "构建工具", "脚手架"]
description = "用一个小项目讲清楚 entry/output/loaders/plugins 等构建基础概念。"
+++

那会儿刚开始上手 Webpack 的时候，看官方文档一堆术语：entry、output、loader、plugin，看得云里雾里。直到自己从零搭了一套非常简陋的脚手架，才真正把这些概念串起来。

这篇文章就用一个「最小可用前端脚手架」为例，帮你把 Webpack 入门阶段最常见的概念打通。

---

## 目标：一个最小可用的前端脚手架

希望做到：

- 支持打包 ES6 代码（`import`、`箭头函数` 等）。
- 支持打包 CSS。
- 支持本地开发服务器 + 热更新。
- 打包输出一份 `dist`，可以直接丢到静态服务器上跑。

目录长这样：

```bash
my-webpack-starter
├── package.json
├── webpack.config.js
├── /src
│   ├── index.js
│   └── index.css
└── /dist  # 构建后生成
```

---

## 第一步：初始化项目

```bash
mkdir my-webpack-starter
cd my-webpack-starter
npm init -y
```

安装 Webpack 相关依赖：

```bash
npm install webpack webpack-cli webpack-dev-server --save-dev
```

---

## 第二步：准备一个简单入口文件

`src/index.js`：

```js
import './index.css';

const root = document.getElementById('app');
root.innerText = 'Hello Webpack Starter!';
```

`src/index.css`：

```css
body {
  margin: 0;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

#app {
  padding: 24px;
  font-size: 20px;
}
```

`public/index.html`（作为模板）：

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>Webpack Starter</title>
</head>
<body>
  <div id="app"></div>
</body>
</html>
```

---

## 第三步：写 webpack.config.js

先安装一些常用 loader / 插件：

```bash
npm install style-loader css-loader --save-dev
npm install html-webpack-plugin --save-dev
```

新建 `webpack.config.js`：

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  // 入口
  entry: './src/index.js',

  // 输出
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.[hash].js',
    clean: true, // 每次构建前清理 dist（Webpack 5）
  },

  // 模式：开发/生产
  mode: 'development',

  // 模块规则：告诉 webpack 遇到什么文件用什么处理
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          'style-loader', // 把 CSS 通过 <style> 标签插入页面
          'css-loader',   // 让 JS 可以 import CSS
        ],
      },
    ],
  },

  // 插件
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html', // 使用我们刚才写的模板
    }),
  ],

  // 开发服务器配置
  devServer: {
    port: 8080,
    hot: true,
    open: true,
  },
};
```

几个关键点：

- **entry**：Webpack 的入口文件，从这里开始构建依赖图。
- **output**：构建结果放到 `dist` 目录，文件名带 hash 方便缓存。
- **module.rules**：告诉 Webpack 遇到 `.css` 文件用 `css-loader` 编译，再用 `style-loader` 插入到页面。
- **plugins**：HtmlWebpackPlugin 自动生成 `index.html` 并注入打包出来的 JS。
- **devServer**：本地开发服务器的端口、是否自动打开浏览器等。

---

## 第四步：在 package.json 里加命令

`package.json`：

```json
{
  "name": "my-webpack-starter",
  "version": "1.0.0",
  "scripts": {
    "dev": "webpack-dev-server --config webpack.config.js",
    "build": "webpack --config webpack.config.js"
  }
}
```

现在就可以：

```bash
npm run dev   # 启动本地开发环境
npm run build # 构建生产包
```

---

## Webpack 核心概念，用这个项目串一下

### 1. Entry（入口）

入口是构建的起点，通常是你的 SPA 入口文件：

```js
entry: './src/index.js',
```

也可以写成多入口：

```js
entry: {
  main: './src/index.js',
  admin: './src/admin.js',
},
```

适合多页面应用，最终生成多个 bundle。

---

### 2. Output（输出）

输出配置决定打包结果放到哪里、叫什么名字：

```js
output: {
  path: path.resolve(__dirname, 'dist'),
  filename: '[name].[contenthash].js',
}
```

- `[name]` 表示 entry 的名称，比如 `main`、`admin`。
- `[contenthash]` 根据内容生成 hash，方便浏览器缓存。

---

### 3. Loader：把各种资源「变成 JS 能理解的东西」

Webpack 只理解 JavaScript 和 JSON，其他类型需要 Loader。

- `css-loader`：把 CSS 文件解析成 JS 能理解的模块。
- `style-loader`：把样式通过 `<style>` 标签插到 HTML 中。
- 类似还有 `babel-loader`、`file-loader`、`url-loader` 等等。

一个典型 rule：

```js
{
  test: /\.css$/,
  use: ['style-loader', 'css-loader'],
}
```

`use` 的执行顺序是**从后往前**，即先 `css-loader`，再 `style-loader`。

---

### 4. Plugin：扩展 Webpack 能力

插件可以在构建过程的各个阶段插入逻辑，例如：

- HtmlWebpackPlugin：生成 HTML 文件并注入资源。
- DefinePlugin：定义环境变量。
- MiniCssExtractPlugin：把 CSS 抽离成独立文件。

我们刚用的：

```js
plugins: [
  new HtmlWebpackPlugin({
    template: './public/index.html',
  }),
],
```

就是在构建结束后，帮你生成一个带 `<script src="bundle.xxx.js">` 的 `index.html`。

---

## 再进一步：加上 Babel 支持 ES6+

通常我们还会再加一层 Babel，把 ES6+ 转成浏览器更容易兼容的代码。

安装：

```bash
npm install @babel/core @babel/preset-env babel-loader --save-dev
```

新增 `.babelrc`：

```json
{
  "presets": ["@babel/preset-env"]
}
```

在 `webpack.config.js` 的 rules 里多加一条：

```js
{
  test: /\.m?js$/,
  exclude: /node_modules/,
  use: {
    loader: 'babel-loader',
  },
},
```

这样你就可以放心写 ES6+，交给 Babel 和 Webpack 来处理。

---

## 小结：脚手架不一定要多「高级」，先跑起来最重要

这篇文章的目标不是讲 Webpack 的所有细节，而是帮你建立一个非常简单的「心智模型」：

- Webpack 的本质：**把各种资源看成模块，构建成一个依赖图，然后输出适合浏览器执行的结果。**
- Entry / Output 决定「从哪来、到哪去」。 
- Loader 把非 JS 资源变成 JS 模块。
- Plugin 在构建流程的各个阶段插入额外能力。
- 搭好最小的脚手架之后，再去慢慢加优化（代码分割、缓存策略、压缩等）会更有感觉。

当你真的能从零把这个 starter 跑起来，再去看官方文档，会顺畅很多。
