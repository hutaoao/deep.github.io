---
layout: post
title: "JavaScript 模块化演进史"
date: 2026-02-03 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: ["模块化", "IIFE", "CommonJS", "AMD", "UMD", "ESModule", "面试"]
---

## 一句话概括

JS 模块化十年进化史，每一代都在解决上一代最痛的伤疤：IIFE（没封装）→ CJS（没异步）→ AMD（写起来啰嗦）→ UMD（格式割裂）→ ESM（编译时没信息）。理解这段历史，你就理解了为什么 `import` 是今天这个样子。

## 核心知识点

### 1. IIFE — 用函数作用域造模块（2009 年以前）

```javascript
// 困境：全局变量满天飞，jQuery 占了 $，Prototype 也占 $
// <script src="jquery.js"></script>
// <script src="myPlugin.js"></script>  ← 顺序错了就炸

// IIFE 解法：闭包锁住私有变量，只暴露接口
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

**解决了：** 封装。**没解决：** 依赖管理——文件加载顺序依然靠 `<script>` 标签手动排列，10 个文件以上就是排雷游戏。

### 2. CommonJS — Node.js 的同步模块（2009）

```javascript
// 思路：读文件 → 跑代码 → 缓存结果 → 返回 exports
const fs = require('fs');
module.exports = { doWork() {} };
```

**解决了：** 明确的依赖关系图和模块封装。**致命缺陷：** `require` 用 `readFileSync` 是同步操作——Node 从磁盘读当然快，但浏览器从网络下载是异步的，同步 `require` 会把整个页面卡死。

### 3. AMD — 浏览器原生异步（2011，RequireJS）

```javascript
// 依赖前置声明，异步并行下载，全到位后再执行
define('myModule', ['jquery', 'lodash'], ($, _) => ({
  init: () => $('.app').text(_.capitalize('hello'))
}));
require(['myModule'], my => my.init());
```

**解决了：** 浏览器异步加载——并行下载依赖，不卡页面渲染。**代价：** 每个模块一层 `define` + callback，嵌套一多像洋葱皮。

### 4. UMD — 一个文件通吃所有环境（过渡方案）

```javascript
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
    define(['jquery'], factory);                     // AMD 环境
  } else if (typeof module === 'object' && module.exports) {
    module.exports = factory(require('jquery'));     // CJS 环境
  } else {
    root.MyLib = factory(root.jQuery);               // 浏览器全局
  }
})(this, ($) => { /* 业务逻辑 */ });
```

**思路：** 运行时检测当前环境 → 选对应的模块格式。Lodash、Moment.js 等老牌库的文件头几乎全是这段话。现代新库基本只输出 ESM，UMD 已退役。

### 5. ES Module — 语言级别的终极答案（ES2015+）

```javascript
// 静态 + 异步 + 语言内建
export const add = (a, b) => a + b;
import { add } from './math.js';

// 动态按需加载
const lazy = await import('./heavy.js');
```

| 进化节点 | 解决的痛点 | 引入的新问题 |
|---|---|---|
| IIFE | 封装 → 没有全局污染 | 依赖顺序靠人工排 |
| CommonJS | 依赖关系图 | 同步加载不适合浏览器 |
| AMD | 异步加载 | 写法啰嗦，一层 callback |
| UMD | 格式兼容 | 代码冗余，运行时判断 |
| ES Module | 静态分析 + 浏览器原生 | Node 端兼容过渡期长 |

## 其实你每天都在用

- **npm 包双入口**：`"main": "index.js"` (CJS) + `"module": "index.mjs"` (ESM)，打包工具自动选最优的
- **Webpack `__webpack_require__`**：在浏览器里山寨了一套 CJS 运行时，让你写 `require` 风格的代码也能跑
- **Vite 开发时零打包**：浏览器原生 `<script type="module">` 直接跑 ESM，100+ 模块也是浏览器自己按需 fetch
- **babel/tsc 转译 import → require**：你代码里写的是 ESM，实际跑在 Node 里被转成了 CJS
- **`import()` 代码分割**：每次写路由懒加载背后都是动态 import 在切 chunk

## 常见误解

- **❌ 误区：「模块化就是防止全局变量污染」** 这只是第一步（IIFE 时代就解决了）。模块化的核心是**显式依赖管理**——一万行代码后你能一眼看出 A 模块依赖谁、被谁依赖。IIFE 有封装但没有依赖图。

- **❌ 误区：「AMD 已经入土了，了解没意义」** RequireJS 确实死了，但 AMD 的核心理念——依赖前置声明、并行异步下载——在 `import()` 和 HTTP/2 多路复用时代重新焕发了生命力。了解 AMD 才懂为什么 ESM "静态+动态" 双模式设计如此精妙。

- **❌ 误区：「UMD 纯历史包袱」** 在 ESM 还没普及的年代，写一个库需要同时兼容 `<script>` 标签、CJS、AMD 时 UMD 是唯一解。今天新库基本只输出 ESM，UMD 确实可以进博物馆了。

- **❌ 误区：「babel 转译后 CJS 和 ESM 就是等价的了」** babel 能转语法（`import` → `require`），但**转不了语义**：Live Binding（实时绑定）和值拷贝的区别转译不出来。Webpack 额外注入了 getter 来模拟 live binding，但这层抽象有时会引入微妙差异。

## 一句话总结

ES Module 不是某一天突然从天而降的完美方案——它站在 IIFE 的封装、CJS 的依赖管理、AMD 的异步加载三座肩膀上。理解每一代的痛和解决思路，才能真正理解 `import/export` 的设计意图。
