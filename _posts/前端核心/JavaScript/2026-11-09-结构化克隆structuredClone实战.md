---
layout: post
title: "结构化克隆 structuredClone 实战"
date: 2026-11-09 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [structuredClone, 深拷贝, 结构化克隆, DataCloneError, Transferable, 循环引用]
description: >
  structuredClone 是 2022 年起浏览器/Node 原生支持的深拷贝函数，底层同 postMessage/IndexedDB，面试常考它和 JSON 方案、手写深拷贝的差异与边界。
---

## 一句话概括

`structuredClone()` 是 2022 年起浏览器 / Node 原生支持的**深拷贝**全局函数，底层就是 `postMessage`、IndexedDB 用的那套结构化克隆算法。面试官问它，核心是考三点：**①它和 `JSON.parse(JSON.stringify())` 到底差在哪；②哪些类型它支持、哪些会直接抛 `DataCloneError`；③它"深拷贝但不拷贝行为"的设计哲学**。记住一句话：能拷贝数据、不拷贝函数；能保留循环引用、丢原型链。

## 核心知识点

### 1. 基本用法：一行深拷贝，还支持循环引用

```js
const original = { a: 1, b: { c: 2 } };
const copy = structuredClone(original);
copy.b.c = 100;
console.log(original.b.c); // 2，互不影响，是真深拷贝

// 循环引用也不会崩
const obj = {};
obj.self = obj;
const clone = structuredClone(obj);
console.log(clone.self === clone); // true，循环结构被完整保留
```

对比老办法 `JSON.parse(JSON.stringify())`：遇到循环引用直接 `TypeError: Converting circular structure to JSON`。这是 `structuredClone` 最直观的胜利。

### 2. 它 vs JSON 方案：类型保全是最大差异

这是面试必背的对比表：

| 特性 | JSON.parse(JSON.stringify()) | structuredClone() |
|---|---|---|
| 深拷贝 | ✅ | ✅ |
| Date | ❌ 变字符串 | ✅ 保留 |
| Map / Set | ❌ 变 `{}` | ✅ 保留 |
| RegExp | ❌ 变 `{}` | ✅ 保留（lastIndex 不保留） |
| ArrayBuffer / TypedArray | ❌ 丢失 | ✅ 保留 |
| 循环引用 | ❌ 抛错 | ✅ 支持 |
| 函数 / Symbol / undefined | ❌ 静默丢弃 | ❌ 抛 DataCloneError |
| NaN / Infinity | ❌ 变 null | ✅ 保留 |
| 原型链 | ❌ 丢失 | ❌ 也丢失 |

结论：**JSON 方案赢在"到处能跑"，输在"偷改 / 丢类型"**；`structuredClone` 赢在"保类型"，但遇到函数 / Symbol / DOM 直接抛错而不是静默丢。

### 3. 哪些类型支持 / 哪些直接抛 DataCloneError（必背）

支持：Array、ArrayBuffer、Boolean、DataView、Date、**Error 系列**（仅 Error / EvalError / RangeError / ReferenceError / SyntaxError / TypeError / URIError）、Map、Number、普通 Object、原始类型（除 symbol）、RegExp、Set、String、TypedArray，以及 Blob / File / ImageData 等 Web 类型。

```js
// ❌ 这些会直接抛 DataCloneError
structuredClone({ fn: () => {} });              // 函数不行
structuredClone({ sym: Symbol("x") });          // Symbol 不行
structuredClone(document.body);                 // DOM 节点不行
structuredClone(new WeakMap());                 // WeakMap / WeakSet 不行
```

### 4. 它"深拷贝但不拷贝行为"：原型链、getter、类实例都丢

很多人的盲区：它是**数据克隆，不是对象克隆**。

```js
class User {
  constructor(name) { this.name = name; }
  greet() { return "hi " + this.name; }   // 方法不会跟着走
}
const u = new User("Ada");
const clone = structuredClone(u);
console.log(clone instanceof User);  // false —— 原型链没复制，变成普通对象
console.log(clone.greet);            // undefined —— 方法没了

// getter 会被"求值"而不是"复制"
const o = { get computed() { return 42; } };
structuredClone(o).computed;         // 42，但它是个普通数据属性，不再是 getter
```

面试金句：**structuredClone 只拷贝"数据"，不拷贝"行为"**——函数、原型、getter / setter、属性描述符、Symbol 键一概不保留。类实例会被拍平成一个普通对象。

### 5. transfer：把可转移对象"搬"过去而不是复制

第二个参数 `{ transfer: [...] }` 可以把 Transferable（ArrayBuffer、MessagePort 等）**转移**到新对象，原对象被"剥离"（detached），不能再访问：

```js
const buffer = new ArrayBuffer(16);
const transferred = structuredClone(buffer, { transfer: [buffer] });
console.log(buffer.byteLength);  // 0 —— 原 buffer 被清空、不可用了
```

适用场景：异步校验缓冲区数据前想先克隆一份，并阻止别人改原 buffer。注意**转移 ≠ 克隆**，原对象会失效。

### 6. 兼容性 & 兜底

Baseline 已广泛支持（Chrome 98 / Edge 98 / Firefox 94 / Safari 15.4 起，Node 17+）。老环境兜底：

```js
function deepClone(val) {
  return typeof structuredClone === "function"
    ? structuredClone(val)
    : JSON.parse(JSON.stringify(val));  // 兜底，但功能弱（丢类型）
}
```

### 7. 和 Vue reactive 一起用的坑

直接 `structuredClone(reactiveObj)` 不会抛错——克隆算法会**读穿 Proxy** 拿到底层数据，但结果是一个**失去响应式的纯普通对象**。所以它可以当"取一份快照数据"用，但别指望克隆出的对象还有响应式；而且如果 reactive 对象里挂了函数 / Symbol，照样抛 `DataCloneError`。

## 其实你每天都在用

- **状态管理里做不可变更新**：Redux reducer 要求返回新对象，用 `structuredClone` 拿独立副本再改，避免改到原引用
- **深拷贝配置 / 表单草稿**：编辑草稿时克隆一份原始配置，取消就丢弃，互不影响
- **postMessage / Web Worker / IndexedDB**：底层全靠同一套结构化克隆算法，你早就在用它了
- **组件 props 拷贝**：Vue 里 `copy.value = structuredClone(props.data)` 避免父子共用引用导致的意外修改
- **脱离响应式的纯数据快照**：React / Vue 里想拿一份"不被框架追踪"的数据时

## 常见误解（FAQ）

**❌ 误区一："structuredClone 就是 JSON 方案的升级版，随便换"**

方向对，但行为差很大。JSON 遇函数 / Symbol / undefined 是**静默丢弃**，你可能在不知不觉丢了数据；`structuredClone` 遇函数 / Symbol / DOM 是直接**抛 DataCloneError**，逼你立刻发现。而且 JSON 会偷改类型（Date → 字符串、Map → {}），structuredClone 不会。两者错误处理风格完全不同。

**❌ 误区二："它能克隆任何东西，包括函数和 DOM"**

直接抛 `DataCloneError`。它只拷"可序列化数据"，函数、Symbol、DOM 节点、WeakMap / WeakSet 全部不可克隆。想拷带方法的对象，还是得手写深拷贝或 lodash。

**❌ 误区三："克隆类实例能得到一个一模一样的实例"**

类实例会被**拍平成普通对象**——原型链断了，`instanceof` 变 false，方法没了。它只复制可枚举的数据属性。需要保留类结构，得自己写深拷贝并手动恢复原型。

**❌ 误区四："循环引用它会栈溢出"**

不会。结构化克隆算法内部维护了"已访问引用"的 map，专门防无限递归，`obj.self = obj` 这种循环结构能正确保留。倒是手写递归深拷贝如果不处理环才会栈溢出。

**❌ 误区五："transfer 是复制了一份 buffer"**

正好反了。`transfer` 是**转移所有权**：原对象被 detach（byteLength 变 0、不可再用），数据被"搬"到新对象。想保留原对象，就别传 transfer，让它走正常克隆。

## 一句话总结

`structuredClone()` 是浏览器原生深拷贝，底层同 postMessage / IndexedDB，赢在**保类型、支持循环引用**，输在**不拷函数 / Symbol / DOM（直接抛错）、丢原型链与行为**；记住"只拷数据不拷行为"，遇到方法 / 类实例就换手写深拷贝。
