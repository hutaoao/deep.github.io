---
layout: post
title: "JavaScript 模块化演进史"
date: 2026-02-03
categories: ["前端核心", "JavaScript"]
tags: ["模块化", "IIFE", "CommonJS", "AMD", "UMD", "ESModule", "面试"]
---

## 一句话概括

JS 模块化十年进化史，每一代都在解决上一代最痛的伤疤：IIFE（没封装）→ CJS（没异步）→ AMD（写起来啰嗦）→ UMD（格式割裂）→ ESM（静态分析需求）。理解这段历史，你就理解了为什么今天的 `import/export` 是这样设计的。

## 核心知识点

### 1. IIFE——用闭包硬造模块（2009 年前）

```js
// 困境：所有变量丢全局，顺序全靠 <script> 标签手动排
// <script src="jquery.js"></script>
// <script src="plugin.js"></script>  ← 顺序错了当场炸

// IIFE 解法：函数作用域锁住私有变量
const MyLib = (() => {
  const secret = '外面碰不到';
  return {
    getSecret: () => secret,
    version: '1.0.0'
  };
})();

MyLib.getSecret(); // '外面碰不到'
MyLib.secret;      // undefined —— 封装达成！
```

**解决了什么：** 封装——不再全丢全局。**没解决什么：** 依赖管理——10 个文件就是排雷游戏，你得记住加载顺序。

### 2. CommonJS——Node.js 的同步模块（2009）

```js
const fs = require('fs');
module.exports = { doWork() {} };
// 思路：读文件 → 跑代码 → 缓存 → 返回 exports，一气呵成
```

**解决了：** 明确的依赖关系图 + 模块封装。**致命伤：** `require` 底层用 `readFileSync` 是同步的——Node 从磁盘读当然快，浏览器从网络下载可不行，同步 `require` 会把整个页面卡死。

### 3. AMD——浏览器端的异步救星（2011，RequireJS）

```js
// 依赖前置声明 → 并行下载 → 全部到位后再执行
define('myModule', ['jquery', 'lodash'], ($, _) => ({
  init: () => $('.app').text(_.capitalize('hello'))
}));
require(['myModule'], my => my.init());
```

**解决了：** 浏览器异步加载——并行下载依赖，不卡渲染。**代价：** 每个模块一层 `define` callback，嵌套多了像剥洋葱。

### 4. UMD——一个文件通吃所有环境（过渡期产物）

```js
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
    define(['jquery'], factory);                     // AMD
  } else if (typeof module === 'object' && module.exports) {
    module.exports = factory(require('jquery'));     // CJS
  } else {
    root.MyLib = factory(root.jQuery);               // 浏览器全局
  }
})(this, ($) => { /* 业务逻辑 */ });
```

Lodash、Moment.js 等老牌库的源码文件头几乎全是这个套路——运行时检测环境选格式。今天新库基本只输出 ESM，UMD 已进博物馆。

### 5. ES Module——语言级别的标准答案（ES2015+）

```js
export const add = (a, b) => a + b;
import { add } from './math.js';

// 动态按需加载
const LazyPage = await import('./heavy.js');
```

| 进化节点 | 解决的痛点 | 引入的新问题 |
|---|---|---|
| IIFE | 封装（不再全局污染） | 依赖顺序靠人工排 |
| CJS | 依赖关系图 | 同步加载不适合浏览器 |
| AMD | 浏览器异步 | 写法啰嗦 |
| UMD | 格式兼容 | 运行时判断 + 代码冗余 |
| ESM | 静态分析 + 浏览器原生 | Node 端兼容过渡期长 |

## 其实你每天都在用

- **npm 包双入口**：`"main": "index.js"` (CJS) + `"module": "index.mjs"` (ESM)，打包工具自动选最优
- **Vite 零打包开发**：浏览器原生 `<script type="module">` 直接跑 ESM，100+ 模块按需 fetch
- **Webpack `__webpack_require__`**：在浏览器里山寨了一套 CJS 运行时
- **Babel / tsc 转译**：你写的是 ESM，Node 里实际跑的是被转成 CJS 的代码
- **`import()` 代码分割**：每次写路由懒加载，底层都是动态 import 在切 chunk

## 常见误解

- **❌ 误区：「模块化就是防止全局变量污染」** 这只是第一步（IIFE 就解决了）。模块化的核心是**显式依赖管理**——一万行代码后你能一眼看出 A 模块依赖谁、被谁依赖。IIFE 有封装但没有依赖图。

- **❌ 误区：「AMD 已入土，了解无意义」** RequireJS 确实死了，但 AMD 的核心理念——依赖前置声明、并行异步下载——在 `import()` 和 HTTP/2 时代重新焕发生命力。了解 AMD 才懂为什么 ESM"静态+动态"双模式设计如此精妙。

- **❌ 误区：「UMD 纯历史包袱」** 当年写库需要同时兼容 `<script>` 标签、CJS、AMD，UMD 是唯一解。今天新库基本只输 ESM，UMD 可以进博物馆了，但看老库源码时你还得认识它。

- **❌ 误区：「Babel 转译后 CJS 和 ESM 就是等价的了」** Babel 能转语法（`import` → `require`），**转不了语义**：Live Binding 和值拷贝的区别靠语法转换实现不了。Webpack 额外注入 getter 来模拟 live binding，但这层抽象偶尔会引入微妙差异。

## 一句话总结

ES Module 不是某天突然降临的完美方案——它站在 IIFE 的封装、CJS 的依赖管理、AMD 的异步加载三座肩膀上。理解每一代的痛和解决思路，才真正理解 `import/export` 为什么是今天这个样子。
