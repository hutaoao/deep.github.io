---
layout: post
title: "函数式工具：手写 curry、compose 与 pipe"
date: 2026-10-26 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [函数式编程, 柯里化, compose, pipe, 高阶函数]
description: >
  面试动手题：手写通用 curry、partial、compose、pipe。讲清原理（fn.length + 闭包）、执行顺序（右到左 vs 左到右）与两个高频易错点。
---

## 一句话概括

上一篇讲了"思想"，这一篇是"动手"。面试常让你现场写三个工具函数：**`curry`**（把多参函数变成可分步调用的柯里化版本）、**`compose`**（把多个函数从右到左组合）、**`pipe`**（从左到右组合）。它们的共同点是"高阶函数 + 闭包"：curry 用闭包收集参数，compose/pipe 用 `reduce` 把函数串成一条流水线。搞清"参数什么时候够、函数按什么顺序执行"，这三类手写题基本就不会翻车。

## 核心知识点

### 1. 手写 curry：核心是 `fn.length` + 闭包收集参数

**结论**：通用 curry 的关键是——用闭包把每次传进来的参数攒起来，当"已收集参数个数 ≥ 原函数形参个数（`fn.length`）"时，就真正调用原函数；否则返回一个继续收集参数的新函数。

```js
function curry(fn) {
  return function curried(...args) {
    // 参数够了就执行原函数（注意用 apply 透传 this）
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    // 不够就返回新函数，把历史参数和新的拼起来继续收
    return function (...rest) {
      return curried.apply(this, args.concat(rest));
    };
  };
}

const add = (a, b, c) => a + b + c;
const curriedAdd = curry(add);
curriedAdd(1)(2)(3); // 6
curriedAdd(1, 2)(3); // 6  —— 通用版支持"一次传多个"或"逐个传"
```

注意 `fn.length` 取的是**形参个数**（不含 rest 参数）。进阶坑见 FAQ 第三条。

### 2. 手写 partial（偏函数）：比 curry 更简单

**结论**：偏函数不需要判断参数够不够，直接"把预设参数拼到前面，返回新函数"。

```js
function partial(fn, ...preset) {
  return (...rest) => fn(...preset, ...rest);
}

const multiply = (a, b, c) => a * b * c;
const doubleAndTriple = partial(multiply, 2, 3); // 固定前两个参数
doubleAndTriple(4); // 2 * 3 * 4 = 24
```

对比 curry：partial 一次能固定多个参数，实现也更短；它本质是 `Function.prototype.bind` 的"手搓版"。

### 3. 手写 compose：从右到左执行

**结论**：`compose(f, g, h)(x)` 等价于 `f(g(h(x)))`——**最右边的函数先执行**，结果一路往左传。实现用 `reduceRight`。

```js
const compose = (...fns) => (x) =>
  fns.reduceRight((acc, fn) => fn(acc), x);

const trim = (s) => s.trim();
const upper = (s) => s.toUpperCase();
const exclaim = (s) => s + '!';

const shout = compose(exclaim, upper, trim);
shout('  hi  '); // 先 trim → upper → exclaim => 'HI!'
```

记忆法：数学里 `f∘g` 就是先 g 后 f，compose 完全对应"右到左"。Redux 的中间件组合就用的是同一套思路（`reduceRight`）。

### 4. 手写 pipe：从左到右执行

**结论**：`pipe` 和 `compose` 唯一区别是**执行顺序反过来**——最左边的函数先执行，像水流一样从左到右流过每个函数。实现用 `reduce`。

```js
const pipe = (...fns) => (x) =>
  fns.reduce((acc, fn) => fn(acc), x);

const shout = pipe(trim, upper, exclaim);
shout('  hi  '); // 先 trim → upper → exclaim => 'HI!'
```

`pipe` 因为符合"从左到右读"的直觉，做数据处理管道时比 compose 更常用。lodash 的 `_.flow` 就是 pipe，`_.flowRight` 就是 compose。

### 5. curry + compose 配合：函数式组合的常见姿势

实际项目里，通常先把 `map/filter` 这类函数柯里化，再用 `pipe` 串成管道：

```js
const map = curry((fn, arr) => arr.map(fn));
const filter = curry((fn, arr) => arr.filter(fn));

const double = (n) => n * 2;
const even = (n) => n % 2 === 0;

// 先过滤偶数，再翻倍
const process = pipe(filter(even), map(double));
process([1, 2, 3, 4]); // [4, 8]
```

这就是"把小函数组合成复杂操作"的函数式精髓：每个环节都是纯的、可独立测试。

### 6. 两个高频易错点

- **单参限制**：上面 `reduce` / `reduceRight` 版的 compose/pipe，管道里的每个函数**只能接收 1 个参数（unary）**。如果想让最左/最右的函数接收多个参数，要换成这种写法（让最外侧函数吃多参）：

```js
// 第一个函数可多参的 pipe
const pipe = (...fns) => (...args) =>
  fns.reduce((f, g) => (...a) => g(f(...a)))(...args);
```

- **this 透传**：手写 curry 时要用 `fn.apply(this, args)`，否则原函数是对象方法（依赖 `this`）时会丢上下文。

## 其实你每天都在用

- **Redux 中间件**：`applyMiddleware` 内部用 `reduceRight` 把中间件组合成一个处理链，就是 compose 的思想。
- **lodash**：`_.flow`（= pipe）、`_.flowRight`（= compose）、`_.curry`、`_.partial` 全是现成轮子。
- **React 事件传参**：`onClick={handle(id)}` 这种写法，本质是把 `handle` 柯里化成"先吃 id、再吃 event"的函数。
- **Ramda**：所有函数默认柯里化，直接 `R.pipe(a, b, c)(data)` 串管道。
- **数据处理流水线**：`pipe(校验, 清洗, 脱敏, 入库)`，一眼看懂数据流向。

## 常见误解（FAQ）

**❌ 误区一："curry 只能 `add(1)(2)(3)` 这样逐个传"**

通用版 curry 支持混合传参：`add(1, 2)(3)`、`add(1)(2, 3)` 都成立，关键判断是 `args.length >= fn.length`。只有"手写死成嵌套箭头函数"那种简易版才强制逐个传——面试写通用版更能拿分。

**❌ 误区二："compose 和 pipe 只是顺序相反，随便用一个"**

很多人当场搞反顺序导致结果错。`compose` 最右先执行，`pipe` 最左先执行——写之前先想清楚"数据从哪进、往哪出"。混淆了顺序，组合出来的函数逻辑就完全反了。

**❌ 误区三："`fn.length` 永远等于参数个数"**

带**默认值**的参数会让 `length` 截断。例如 `function fn(a, b = 1, c) {}` 的 `length` 是 `1`（只统计到第一个带默认值的参数之前）。手写 curry 时如果遇到带默认值的函数，`fn.length` 会偏小、提前触发执行，需要额外传 arity 或用占位符机制处理。

**❌ 误区四："用箭头函数递归写 curry 最简洁"**

纯箭头递归需要给函数命名（具名箭头），可读性往往更差，而且容易忘记 `this` 透传。面试用"具名函数表达式 + `apply` 透传 `this`"的写法更清晰、更稳，也方便讲解。

## 一句话总结

`curry` 靠闭包攒参数（看 `fn.length` 判断是否齐）、`compose` 右到左用 `reduceRight`、`pipe` 左到右用 `reduce`，再把柯里化后的小函数丢进 `pipe` 串成数据流——这就是函数式工具链最核心的四行代码。
