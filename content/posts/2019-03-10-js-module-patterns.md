+++
title = "JavaScript 模块化那些事：从 IIFE 到 ES Modules"
date = 2019-03-10
draft = false
categories = ["前端基础"]
tags = ["JavaScript", "模块化", "ESModules"]
description = "回顾从 jQuery 时代到 ES Modules 的模块化演变，以及现在更推荐的写法。"
+++

最早写前端的时候，经常是一个 `index.html` 里塞一大坨 `<script>`，变量到处飞，函数名一改项目全崩。后来慢慢开始接触各种模块化方案，从 IIFE、AMD、CommonJS 一路走到 ES Modules，心态也经历了从凑合能用到必须有规范。

这篇就当是给自己的一个小总结：**为什么需要模块化、几种常见写法的区别、现在应该怎么写。**

---

## 为什么要模块化？

几个非常现实的问题：

- 全局变量名冲突：不同文件里不小心定义了同名变量/函数。
- 依赖关系混乱：这个文件依赖谁？要先引哪个 `<script>`？
- 代码难以拆分复用：逻辑越写越大，一改就牵一大片。
- 测试困难：无法只「拿出一小块」单独测试。

所以模块化要解决的就是：

> **把代码按职责拆成「小块」 + 明确每块对外暴露什么 + 明确依赖关系。**

---

## 第一阶段：IIFE 模式（自执行函数）

当时 jQuery 时代最常见的一种写法，就是用自执行函数「造一个自己的小世界」。

```js
(function () {
  const API_URL = '/api/todos';

  function fetchTodos() {
    return fetch(API_URL).then(res => res.json());
  }

  function renderTodos(list) {
    // 渲染逻辑...
  }

  window.todoApp = {
    init() {
      fetchTodos().then(renderTodos);
    },
  };
})();
```

特点：

- 用 IIFE 隔离出一个作用域，避免变量污染全局。
- 需要暴露给外面的东西挂在 `window.todoApp` 上。
- 所有代码其实还是在一个文件里，**没有真正意义上的「依赖管理」**。

适合：**小项目 / 简单封装**。  
不适合：多页面、多模块的大型项目。

---

## 第二阶段：AMD / RequireJS

再往后，就出现了 AMD（Asynchronous Module Definition），代表是 RequireJS。

```js
// math.js
define('math', [], function () {
  function add(a, b) {
    return a + b;
  }

  function mul(a, b) {
    return a * b;
  }

  return {
    add,
    mul,
  };
});

// main.js
define(['math'], function (math) {
  console.log(math.add(1, 2));
});
```

特点：

- 使用 `define` 定义模块，使用 `require`/依赖数组引入。
- 支持浏览器端异步加载脚本。
- 写法偏重「框架约定」，不够自然。

现在很少再新项目用 AMD 了，主要在一些老项目或旧库里还能见到。

---

## 第三阶段：CommonJS（Node.js 世界）

Node.js 出现之后，前端第一次大规模接触到「真正的模块」——CommonJS。

```js
// math.js
function add(a, b) {
  return a + b;
}

function mul(a, b) {
  return a * b;
}

module.exports = {
  add,
  mul,
};
```

```js
// main.js
const math = require('./math');

console.log(math.add(1, 2));
```

特点：

- `require` + `module.exports` / `exports`。
- 模块是 **同步加载** 的，天然适合在服务器端使用。
- 在前端想用 CommonJS，通常要借助打包工具（Browserify / Webpack 等）。

优势：

- 写法简单清晰。
- 非常适合 Node 端和工具链代码。

---

## 第四阶段：ES Modules（现在推荐的主角）

ES6 正式引入了 ES Modules 语法，浏览器和打包工具都已经支持得非常好。

```js
// math.js
export function add(a, b) {
  return a + b;
}

export function mul(a, b) {
  return a * b;
}

// 默认导出
const PI = 3.14;
export default PI;
```

```js
// main.js
import PI, { add, mul } from './math.js';

console.log(add(1, 2));
console.log(PI);
```

特点：

- 语法层面原生支持：`import` / `export`。
- 静态分析友好：打包工具可以基于静态导入做 Tree Shaking。
- 规范统一：前端、Node（在 ESM 模式下）、工具链趋向统一。

浏览器原生支持 `<script type="module">`：

```html
<script type="module">
  import { add } from './math.js';

  console.log(add(1, 2));
</script>
```

---

## ES Modules 的几种用法小结

### 命名导出

```js
// utils.js
export function formatDate(date) {}
export function formatMoney(num) {}
```

```js
// 使用
import { formatDate, formatMoney } from './utils';
```

适合：**导出多个工具函数或常量**。

---

### 默认导出

```js
// apiClient.js
export default class ApiClient {
  // ...
}
```

```js
// 使用
import ApiClient from './apiClient';
```

适合：**一个模块只有一个“主要东西”时**，比如一个类 / 单例对象。

---

### 组合使用

```js
// math.js
export function add(a, b) {}
export function mul(a, b) {}
const PI = 3.14;
export default PI;
```

```js
import PI, { add, mul } from './math';
```

建议：

- 一个文件里不要默认导出太多「意义不清晰」的内容。
- 如果模块是一个「工具集合」，更倾向只用命名导出。

---

## 模块拆分的几个实用建议

1. **按职责拆，不要按文件大小拆**

   - `utils.js` 太大，就拆成 `date.ts` / `string.ts` / `number.ts`。
   - 「接口 + 实现」可以放在一起，让模块自洽。

2. **入口文件要「说人话」**

   做 SDK 或组件库时，通常会有一个入口：

   ```js
   // index.ts
   export * from './api/user';
   export * from './api/order';
   export * from './constants';
   ```

   使用方只需要：

   ```js
   import { getUserInfo } from '@my/sdk';
   ```

3. **避免循环依赖**

   - A 引用 B，B 又引用 A，很容易在运行时出问题。
   - 一旦发现模块之间相互引用太多，可以抽一层「中间模块」出来。

---

## 总结：现在怎么写比较稳妥？

在 2019 之后的新项目里，我的默认选择基本是：

- **浏览器端业务代码：ES Modules**（配合打包工具）。
- **Node / 工具链代码：可以根据项目统一选择 CommonJS 或 ESM**，保持一致即可。
- 尽量避免历史包袱：AMD 这类只在维护老项目时碰一下。

一句话总结：

> **用 ES Modules 写干净的模块边界，比纠结语法更重要。模块拆得越清晰，项目越耐用。**
