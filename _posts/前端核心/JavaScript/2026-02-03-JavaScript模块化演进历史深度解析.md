---
layout: post
title: "JavaScript 模块化演进史"
date: 2026-02-03 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: ["模块化", "IIFE", "CommonJS", "AMD", "UMD", "ESModule", "面试"]
---

## 一句话概括

JS 模块化从全局变量大乱斗到 ESM 一统天下，经历了五次进化：IIFE（封装）、CommonJS（服务端）、AMD（异步）、UMD（兼容）、ES Module（语言级）——每次迭代都在解决前一代最致命的问题。

## 核心知识点

### 1. IIFE — 函数作用域模拟模块（2009）

```javascript
// 只有一个全局变量 MyLib，内部变量外部拿不到
const MyLib = (() => {
  const secret = '内部变量，外面访问不了';
  return {
    getSecret: () => secret,
    version: '1.0.0'
  };
})();

MyLib.getSecret(); // '内部变量'
MyLib.secret;      // undefined — 私有！
```

**解决了封装，没解决依赖管理。** 多个 `<script>` 标签的加载顺序完全靠人工——jQuery 要在插件之前、插件要在业务代码之前……人都麻了。

### 2. CommonJS — 服务端同步加载（2009 Node.js）

```javascript
// 读文件、跑代码、缓存结果、返回 exports — 就是这么简单
const fs = require('fs');
const utils = require('./utils');
module.exports = { doWork() { /* ... */ } };
```

**致命缺陷：** `require` 是同步的。浏览器里从网络下载文件是异步的，同步 `require` 会卡死整个页面。

### 3. AMD — 浏览器天生异步（2011 RequireJS）

```javascript
// 依赖前置声明，异步并行下载
define('myModule', ['jquery', 'lodash'], ($, _) => ({
  init: () => $('.app').text(_.capitalize('hello'))
}));

require(['myModule'], my => my.init());
```

解决了浏览器异步加载，但写法啰嗦——一个模块套一层 `define` + callback。

### 4. UMD — 兼容一切的胶水层（过渡方案）

```javascript
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
    define(['jquery'], factory);           // AMD
  } else if (typeof module === 'object') {
    module.exports = factory(require('jquery')); // CJS
  } else {
    root.MyLib = factory(root.jQuery);     // 全局变量
  }
})(this, $ => { /* 业务逻辑 */ });
```

**一个文件，三种环境通吃。** Lodash、Moment.js 等老牌库的文件头全是这玩意。

### 5. ES Module — 语言级终极答案（ES2015+）

```javascript
// 静态、异步、语言内建 — 三杀
export const add = (a, b) => a + b;
import { add } from './math.js';
const lazy = await import('./heavy.js');
```

| 进化节点 | 解决的痛点 |
|---|---|
| IIFE | 封装 → 摆脱全局变量污染 |
| CommonJS | 依赖管理 → 明确 require 关系 |
| AMD | 异步加载 → 浏览器不卡死 |
| UMD | 格式兼容 → 一套代码多环境 |
| ES Module | 静态分析 → Tree Shaking + 浏览器原生 |

## 其实你每天都在用

- **npm 包的双入口**：`"main"` 指 CJS，`"module"` 指 ESM，打包工具自动选择
- **Webpack 的 `__webpack_require__`**：山寨版 require，在浏览器模拟 CJS
- **Vite 开发时不打包**：浏览器原生 `<script type="module">` 直接跑 ESM，体验秒杀 Webpack
- **babel / tsc 转译**：你写的 ESM `import` 被转成 CJS `require` 才能跑在 Node 里
- **Node.js 的 `.mjs` 和 `"type": "module"`**：让服务端也能原生 ESM

## 常见误解

- **❌ 误区：「模块化就是为了防止全局变量污染」** 这只是第一步。模块化的核心价值是**依赖管理**——上万行代码后，你能不能一眼看出 A 模块依赖谁、被谁依赖。IIFE 解决了封装，但没解决这个问题。

- **❌ 误区：「AMD 已经死了，了解它没意义」** RequireJS 确实死了，但 AMD 的异步加载理念在 `import()` 中复活。了解 AMD 才能理解为什么 ESM 的"静态 + 动态"双模式设计这么精妙。

- **❌ 误区：「UMD 是历史包袱」** 在 ESM 普及前，如果你想写一个库同时被 `<script>` 标签、CJS、AMD 三方引用，UMD 是唯一答案。今天的新库基本只输出 ESM 了，UMD 已退出现役。

- **❌ 误区：「CJS 和 ESM 混用靠 babel 就行」** babel 解决语法转换，但 `require` 和 `import` 的运行语义（值拷贝 vs 实时绑定）转译不了。Webpack 为 ESM → CJS 做了特殊处理来模拟 live binding，但这层抽象有时会引入微妙的差异。

## 一句话总结

JS 模块化的历史就是"封装 → 依赖管理 → 异步加载 → 格式统一 → 静态优化"的迭代链——ES Module 不是突然冒出来的天降猛男，它站在 IIFE、CommonJS 和 AMD 的肩膀上。
