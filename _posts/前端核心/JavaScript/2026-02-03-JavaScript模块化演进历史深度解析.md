---
layout: post
title: "JavaScript 模块化演进史"
date: 2026-02-03 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: ["模块化", "IIFE", "CommonJS", "AMD", "UMD", "ESModule"]
---

## 一句话概括

JS 模块化经历了从「脚本地狱」（全局变量大乱斗）到 IIFE → CommonJS → AMD → UMD → ES Module 的五次进化，每次迭代都在解决前一代的致命缺陷。

## 核心知识点

### 1. IIFE 时代 — 用函数作用域模拟封装

```javascript
// 1999年的"模块"长这样
const MyLib = (function () {
  const secret = '内部变量，外面拿不到';
  return {
    getSecret: () => secret,
    version: '1.0.0',
  };
})();

console.log(MyLib.getSecret()); // '内部变量'
console.log(MyLib.secret);      // undefined — 私有！
```

**解决了封装，但没解决依赖管理。** 多个 script 标签的加载顺序依然靠人工保证。

### 2. CommonJS — 服务端模块化标准

```javascript
// Node.js 的思路：同步读文件 + 模块缓存
const fs = require('fs');
const utils = require('./utils');
module.exports = { doWork() { /* ... */ } };
```

**致命缺陷：** `require` 是同步的。在浏览器里，网络下载文件是异步的，同步 `require` 会卡死 UI。

### 3. AMD — 浏览器天生异步

```javascript
// RequireJS 时代的写法
define('myModule', ['jquery', 'lodash'], function ($, _) {
  return { init: () => $('.app').text(_.capitalize('hello')) };
});

require(['myModule'], function (my) { my.init(); });
```

依赖前置声明、异步并行下载。**问题：写法太啰嗦，回调地狱初现。**

### 4. UMD — 兼容一切的胶水层

```javascript
// jQuery / Lodash 的经典 UMD 模板
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
    define(['jquery'], factory);           // AMD
  } else if (typeof module === 'object') {
    module.exports = factory(require('jquery')); // CommonJS
  } else {
    root.MyLib = factory(root.jQuery);     // 全局变量
  }
})(this, function ($) { /* 业务逻辑 */ });
```

**过渡方案，写入一个文件后三种环境都能跑。** 现在不需要了——ESM 统一天下。

### 5. ES Module — 语言级终极方案

```javascript
// 静态、异步、语言内建 — 三大杀器
export const add = (a, b) => a + b;
import { add } from './math.js';
const lazy = await import('./heavy.js');
```

**对比 CJS：**

| 维度 | CommonJS | ES Module |
|------|----------|-----------|
| 语法 | `require` / `module.exports` | `import` / `export` |
| 时机 | 运行时 | 编译时 |
| 导出 | 值拷贝 | **动态绑定**（引用） |
| Tree Shaking | ❌ | ✅ |
| 适用 | Node.js | 浏览器 + Node.js |

## 其实你每天都在用

- **npm 包的双入口**：`package.json` 里 `"main"` 指向 CJS，`"module"` 指向 ESM，打包工具自动选
- **Webpack 的 `__webpack_require__`** 就是山寨版 require，让你在浏览器里跑 CJS
- **Vite 不打包直接跑 ESM**：利用浏览器原生 `<script type="module">`，开发体验直接起飞
- **babel 转译 import → require**：很多项目源码写 ESM，但跑在 Node 里靠 babel 转成 CJS
- **Node.js 的 `.mjs` 和 `"type": "module"`**：让 Node 也能原生跑 ESM

## 常见误解

- **❌ 误区：「模块化就是为了防止全局变量污染」** 这只是第一步。模块化的核心价值是**依赖管理**——代码量上万行后，你能不能一眼看出模块 A 依赖了什么、被谁依赖。IIFE 做到了封装，但没做到这一点。

- **❌ 误区：「AMD 已经被淘汰了，不用了解」** RequireJS 确实死了，但 AMD 的异步加载理念在 `import()` 动态导入中复活了。了解 AMD 能帮你理解为什么 ESM 的静态+动态双模式设计这么精妙。

- **❌ 误区：「UMD 是历史包袱」** 在 ESM 普及前，如果你写了一个库想同时被 `<script>` 标签、CJS、AMD 引用，UMD 是唯一解。这就是为什么 Lodash、Moment.js 等老牌库的文件头里都有一大段 UMD 判定代码。

- **❌ 误区：「CJS 和 ESM 混用只需要 babel 就行」** babel 解决的是语法转换，但 `require` 和 `import` 的**运行语义**（值拷贝 vs 实时绑定）转译不了。Webpack 对 ESM → CJS 做了特殊处理来模拟 live binding，但这层抽象有时会引入微妙的行为差异。

## 一句话总结

JavaScript 模块化的演变史，就是**封装 → 依赖管理 → 异步加载 → 静态优化**的逐步升级——ES Module 不是突然冒出来的，它站在 IIFE、CommonJS、AMD 的肩膀上。
