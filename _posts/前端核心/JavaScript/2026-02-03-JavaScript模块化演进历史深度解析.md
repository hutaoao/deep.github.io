---
layout: post
title: "JavaScript 模块化演进史深度解析"
description: >
  从 IIFE 到 CJS、AMD、UMD 再到 ESM，每一代解决上一代最痛的点（封装、异步、写法、格式割裂、静态分析）。
  面试讲清演进脉络，就能说明 import/export 为何是今天这个样子。
date: 2026-02-03
categories: ["前端核心", "JavaScript"]
tags: ["模块化", "IIFE", "CommonJS", "AMD", "UMD", "ESModule", "面试"]
---

## 一句话概括

JS 模块化十年进化，每一代都在解决上一代最痛的伤疤：IIFE（没封装）→ CJS（没异步）→ AMD（写起来臃肿）→ UMD（格式割裂）→ ESM（需要静态分析）。理解这段历史，你就理解了今天的 `import/export` 为什么是今天这个样子。

## 核心知识点

### 1. 史前时代——全局变量 + `<script>` 顺序地狱（2009 前）

```js
// 困境：所有变量丢 window，依赖顺序全靠 script 标签手动排
// <script src="jquery.js"></script>
// <script src="plugin.js"></script>  ← 顺序错了当场炸
```

**核心痛点：** 封装不存在，依赖管理靠人类记忆——10 个文件就是排雷游戏，100 个文件直接放弃治疗。

### 2. IIFE — 用闭包硬造模块（2009 前）

```js
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

**解决了：** 封装——私有变量不再裸奔在全局。**没解决：** 依赖管理——10 个 IIFE 文件还是得手动排顺序。

### 3. CommonJS — Node.js 的同步模块（2009）

```js
const fs = require('fs');
module.exports = { doWork() {} };
// 流程：读文件 → 跑代码 → 缓存 exports → 返回，一气呵成
```

**解决了：** 明确的依赖关系图 + 模块封装。**致命伤：** `require()` 底层 `readFileSync` 是同步的——Node 从磁盘读当然快，浏览器从网络下载同步阻塞会把整个页面卡死。

### 4. AMD — 浏览器端的异步救星（2011，RequireJS）

```js
// 依赖前置声明 → 并行下载 → 全部到位后再执行
define('myModule', ['jquery', 'lodash'], ($, _) => ({
  init: () => $('.app').text(_.capitalize('hello'))
}));
require(['myModule'], my => my.init());
```

**解决了：** 浏览器异步加载——并行下载依赖，不卡页面。**代价：** 每个模块一层 `define` 回调，嵌套深了像剥洋葱。CMD（Sea.js）推崇"用时再 require"来对抗这个问题。

### 5. UMD — 一个文件通吃所有环境（过渡产物）

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

Lodash、Moment.js 等老牌库的源码文件头几乎全是这个套路。今天新库基本只输出 ESM，UMD 已进博物馆。

### 6. ES Module — 语言级标准答案（ES2015+）

| 进化节点 | 解决的痛点 | 引入的新问题 |
|---|---|---|
| IIFE | 封装（不再全局污染） | 依赖顺序靠人工 |
| CJS | 依赖关系图 | 同步加载不适合浏览器 |
| AMD | 浏览器异步加载 | 写法繁琐，回调嵌套 |
| UMD | 多格式兼容 | 运行时判断 + 代码冗余 |
| ESM | 静态分析 + 浏览器原生 | Node 端兼容过渡期长 |

## 其实你每天都在用

- **npm 包双入口**：`"main": "index.js"`（CJS）+ `"module": "index.mjs"`（ESM），打包工具自动选最优
- **Vite 零打包开发**：浏览器原生 `<script type="module">` 直接跑 ESM，上百模块按需 fetch
- **Webpack `__webpack_require__`**：在浏览器里山寨了一套 CJS 运行时，产物里到处可见
- **Babel/tsc 转译**：写的 ESM，Node 里实际跑的是被转成的 CJS
- **`import()` 代码分割**：每次写路由懒加载，底层都是动态 import 在切 chunk
- **React Server Components**：文件顶部 `'use client'` 指令本质上是模块边界标记，继承了 ESM 的静态分析基因

## 常见误解（FAQ）

- **❌ 误区：「模块化就是防止全局变量污染」** 这只是第一步（IIFE 就解决了）。模块化的核心是**显式依赖管理**——一万行代码后你能一眼看出 A 依赖谁、被谁依赖。IIFE 有封装但没有依赖图。

- **❌ 误区：「AMD 已死，了解无意义」** RequireJS 确实退休了，但 AMD 的核心理念——依赖前置声明、并行异步下载——在 `import()` 和 HTTP/2 时代重新焕发了生命力。了解 AMD 才懂为什么 ESM "静态 + 动态"双模式如此精妙。

- **❌ 误区：「UMD 纯历史包袱」** 当年写库需要同时兼容 `<script>` 标签、CJS、AMD 三种环境，UMD 是唯一解。今天新库基本只输出 ESM，但看老库源码时你还得认识它。

- **❌ 误区：「Babel 转译后 CJS 和 ESM 等价了」** Babel 能转语法（`import` → `require`），**转不了语义**：Live Binding 和值拷贝的本质差异靠语法转换实现不了。Webpack 额外注入 getter 来模拟，但这层抽象偶尔会引入微妙差异。

## 一句话总结

ES Module 不是突然降临的完美方案——它站在 IIFE 的封装、CJS 的依赖管理、AMD 的异步加载三座肩膀上。理解每一代的痛和解决思路，才真正理解 `import/export` 为什么是今天这个样子。
