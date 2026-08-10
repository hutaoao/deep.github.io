---
layout: post
title: "手写防抖节流与call/apply/bind深度解析：面试官想听的不止是「背代码」"
date: 2026-07-15 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [防抖, 节流, call, apply, bind, 手写, this]
---

## 一句话概括

防抖和节流是控制高频事件的两把刀：一把让你等用户停手再做（防抖），一把让你按固定节奏做（节流）；而 call/apply/bind 则是手动操控 `this` 的三兄弟，面试官想看到的不只是代码能跑，而是你对 new 优先级、Symbol 临时属性、箭头函数不可改变 this 这些「暗坑」的理解。

## 核心知识点

### 1. 防抖：停手才执行

```js
function debounce(fn, delay = 300) {
  let timer = null
  return function (...args) {
    clearTimeout(timer)
    timer = setTimeout(() => {
      fn.apply(this, args)   // this 用普通函数，保证指向调用者
      timer = null
    }, delay)
  }
}

// 加 leading 和 cancel 的完整版
function debounce2(fn, delay = 300, { leading = false } = {}) {
  let timer = null
  function debounced(...args) {
    if (timer) clearTimeout(timer)
    if (leading && !timer) {
      fn.apply(this, args)  // 第一次立即执行
      timer = setTimeout(() => { timer = null }, delay)
    } else {
      timer = setTimeout(() => {
        fn.apply(this, args)
        timer = null
      }, delay)
    }
  }
  debounced.cancel = () => { clearTimeout(timer); timer = null }
  return debounced
}
```

**关键细节**：返回函数必须是 `function` 而非箭头函数，否则 `this` 指向 `debounce` 的上层而非调用时的上下文。

### 2. 节流：固定频率执行

```js
// 时间戳版 — 第一次立即执行，最后一次不执行
function throttle(fn, interval = 200) {
  let last = 0
  return function (...args) {
    const now = Date.now()
    if (now - last >= interval) {
      last = now
      fn.apply(this, args)
    }
  }
}

// 定时器版 — 第一次延迟执行，最后一次保证执行（可用于 trailing）
function throttle2(fn, interval = 200) {
  let timer = null
  return function (...args) {
    if (!timer) {
      timer = setTimeout(() => {
        fn.apply(this, args)
        timer = null
      }, interval)
    }
  }
}

// 完整版：leading + trailing 都可配
function throttle3(fn, interval = 200, { leading = true, trailing = true } = {}) {
  let timer = null, prev = 0
  return function (...args) {
    const now = Date.now()
    if (!prev && !leading) prev = now
    const remaining = interval - (now - prev)
    if (remaining <= 0) {
      if (timer) { clearTimeout(timer); timer = null }
      fn.apply(this, args)
      prev = now
    } else if (!timer && trailing) {
      timer = setTimeout(() => {
        fn.apply(this, args)
        prev = Date.now()
        timer = null
      }, remaining)
    }
  }
}
```

**判断用哪个**：搜索要结果→防抖；滚动要过程→节流。

### 3. 手写 call：Symbol 当临时钥匙

```js
Function.prototype.myCall = function (context, ...args) {
  // 1. null/undefined → globalThis；基本类型装箱
  context = context ?? globalThis
  if (!['object', 'function'].includes(typeof context)) {
    context = Object(context)
  }
  // 2. Symbol 做临时 key，避免覆盖已有属性
  const key = Symbol('fn')
  context[key] = this
  // 3. 通过隐式绑定执行
  const result = context[key](...args)
  delete context[key]
  return result
}

// 测试
const obj = { name: 'Alice' }
function greet(greeting) { return `${greeting}, ${this.name}` }
greet.myCall(obj, 'Hi')  // "Hi, Alice"
```

**为什么用 Symbol**：直接写 `context.fn = this` 可能覆盖 context 上已有的 `fn` 属性。Symbol 保证唯一。

### 4. 手写 apply：和 call 的区别仅在于参数是数组

```js
Function.prototype.myApply = function (context, args) {
  context = context ?? globalThis
  if (!['object', 'function'].includes(typeof context)) {
    context = Object(context)
  }
  const key = Symbol('fn')
  context[key] = this
  const result = Array.isArray(args) ? context[key](...args) : context[key]()
  delete context[key]
  return result
}
```

### 5. 手写 bind：new 调用时 this 要「叛变」

这是 bind 最常考的坑——bind 返回的函数如果被 new 调用，绑定的 this 会被忽略，this 指向新对象：

```js
Function.prototype.myBind = function (context, ...bindArgs) {
  const fn = this
  const fNOP = function () {}
  const bound = function (...callArgs) {
    // this instanceof fNOP → 说明是 new 调用，this 是刚创建的对象
    return fn.apply(
      this instanceof fNOP ? this : context,
      [...bindArgs, ...callArgs]
    )
  }
  // 维护原型链
  if (fn.prototype) { fNOP.prototype = fn.prototype }
  bound.prototype = new fNOP()
  return bound
}

// 测试：new 调用应覆盖绑定的 this
function Person(name, age) { this.name = name; this.age = age }
const Bound = Person.myBind({ x: 1 }, 'Alice')
const p = new Bound(25)
console.log(p.name)  // 'Alice' (new 覆盖了 { x: 1 })
console.log(p instanceof Person)  // true
```

**原理**：`new bound()` 创建对象 obj，obj 的原型指向 `bound.prototype`，而 `bound.prototype` 是 `new fNOP()` 的实例，所以 `obj instanceof fNOP` 为 true。普通调用 `bound()` 时 this 不继承 fNOP.prototype，所以走 bound 的 context。

## 「其实你每天都在用」

- **搜索框输入联想** — 你打「苹」字后停了 300ms，防抖触发一次请求 → 返回「苹果、苹果手机、苹果醋」。打字快时不发请求
- **窗口 resize 重算布局** — 拖窗口时 resize 事件每秒触发几十次，节流让它每 200ms 才跑一次重算，否则画面像 PPT
- **按钮防重复提交** — 提交订单按钮，用户连续狂点 5 下，节流保持只有一次请求出去（`leading: true` 第一次就发）
- **滚动加载更多** — 列表无限滚动，节流每 200ms 检查一次「是否滚到底」，而不是每像素都触发数据请求
- **call/apply 的隐式使用** — 你写的 `Math.max(...arr)` 没用 apply，但 Babel 编译后的 ES5 代码是 `Math.max.apply(Math, arr)`

## 常见误解

**❌ 误区：「debounce 返回箭头函数就能自动绑定 this」**

恰恰相反——箭头函数的 this 在定义时就锁定为 debounce 调用时的 this（通常是 undefined 或 window），而不是按钮/输入框等的 this。防抖/节流的返回函数**必须用普通 function**。

**❌ 误区：「call/apply/bind 能改变箭头函数的 this」**

箭头函数根本没有自己的 `this`，它的 this 是词法作用域决定的，写在定义时就已经确定。`arrowFn.call(obj)` 只是第一个参数被忽略了——这不是 bug，是 ES6 的设计。

**❌ 误区：「bind 返回的函数如果被 new 调用，绑定的 this 会被覆盖成新对象」**

这个说法是对的——但面试官可能追问：为什么 `this instanceof fNOP` 能判断？本质是 fBound.prototype 通过 `new fNOP()` 创建，new 调用时新对象的原型链上必然有 fNOP.prototype。

**❌ 误区：「throttle 用 setTimeout 就行，不需要时间戳」**

只用 setTimeout 的问题是第一次调用要等一个 interval 才执行（无法实现 leading）。只用时间戳的问题是最后一次触发如果不满足 interval 就永不执行（无法实现 trailing）。完整版需要二者结合。

## 一句话总结

面试官让你手写这些，不是要验证记忆——而是看你是否真正理解 this 绑定的优先级、定时器的异步语义、以及原型链在 new 调用中的作用。背代码只能答前面的 30%，后面 70% 全是边界条件。
