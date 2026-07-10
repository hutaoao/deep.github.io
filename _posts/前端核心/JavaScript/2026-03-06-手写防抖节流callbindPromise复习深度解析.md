---
layout: post
title: "手写防抖节流callbindPromise复习"
date: 2026-03-06
tags: [JavaScript, 防抖, 节流, call, bind, Promise]
categories: [前端核心, JavaScript]
---

## 一句话概括

防抖、节流、call、bind、Promise 是前端面试命中率最高的手写题集合——它们覆盖了闭包、this、异步、微任务等 JS 核心机制，能写出来说明你理解这些 API 到底在底层干了什么。

## 核心知识点

### 1. 防抖（Debounce）—— 等你不抖了再执行

**场景**：搜索输入框每按一个键都发请求 → 用防抖，等停止输入 300ms 再发。

```javascript
function debounce(fn, delay) {
  let timer = null
  return function (...args) {
    clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}

// 进阶版：首次立即执行 (immediate)
function debounce(fn, delay, immediate = false) {
  let timer = null
  return function (...args) {
    const callNow = immediate && !timer
    clearTimeout(timer)
    timer = setTimeout(() => { timer = null; if (!immediate) fn.apply(this, args) }, delay)
    if (callNow) fn.apply(this, args)
  }
}
```

### 2. 节流（Throttle）—— 固定频率执行

**场景**：滚动事件每秒触发几百次 → 节流到每 200ms 执行一次。

```javascript
// 时间戳版 —— 首次立即执行，最后一次不执行
function throttle(fn, delay) {
  let last = 0
  return function (...args) {
    const now = Date.now()
    if (now - last >= delay) {
      fn.apply(this, args)
      last = now
    }
  }
}

// 定时器版 —— 首次延迟执行，最后一次保证执行
function throttle(fn, delay) {
  let timer = null
  return function (...args) {
    if (timer) return
    timer = setTimeout(() => {
      fn.apply(this, args)
      timer = null
    }, delay)
  }
}
```

**怎么区分**：防抖 = 「等你停下再干活」；节流 = 「不管你停不停，我按固定节奏干活」。

### 3. 手写 call / apply / bind

```javascript
// call —— fn.call(ctx, a, b)
Function.prototype.myCall = function (ctx, ...args) {
  const fn = Symbol()
  ctx = ctx == null ? globalThis : Object(ctx)
  ctx[fn] = this
  const result = ctx[fn](...args)
  delete ctx[fn]
  return result
}

// apply —— fn.apply(ctx, [a, b])
Function.prototype.myApply = function (ctx, args) {
  const fn = Symbol()
  ctx = ctx == null ? globalThis : Object(ctx)
  ctx[fn] = this
  const result = ctx[fn](...(args || []))
  delete ctx[fn]
  return result
}

// bind —— 难点：new 绑定优先级 > bind
Function.prototype.myBind = function (ctx, ...bindArgs) {
  const fn = this
  const bound = function (...callArgs) {
    // 如果通过 new 调用，this 指向实例 → 忽略 bind 的 ctx
    return fn.apply(this instanceof bound ? this : ctx, [...bindArgs, ...callArgs])
  }
  bound.prototype = Object.create(fn.prototype)
  return bound
}
```

### 4. Promise（符合 Promises/A+）

```javascript
class MyPromise {
  constructor(executor) {
    this.state = 'pending'
    this.value = undefined
    this.callbacks = []

    const resolve = (val) => {
      if (this.state !== 'pending') return
      this.state = 'fulfilled'
      this.value = val
      this.callbacks.forEach(cb => cb.onFulfilled(val))
    }
    const reject = (err) => {
      if (this.state !== 'pending') return
      this.state = 'rejected'
      this.value = err
      this.callbacks.forEach(cb => cb.onRejected(err))
    }

    try { executor(resolve, reject) } catch (e) { reject(e) }
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const run = () => {
        try {
          const fn = this.state === 'fulfilled' ? onFulfilled : onRejected
          if (typeof fn !== 'function') {
            (this.state === 'fulfilled' ? resolve : reject)(this.value)
            return
          }
          const result = fn(this.value)
          // 返回值是 Promise → 等它 resolve
          if (result instanceof MyPromise) {
            result.then(resolve, reject)
          } else {
            resolve(result)
          }
        } catch (e) { reject(e) }
      }
      // pending 状态暂存回调；已 settled 用微任务执行
      if (this.state === 'pending') {
        this.callbacks.push({ onFulfilled: () => run(), onRejected: () => run() })
      } else {
        queueMicrotask(run)
      }
    })
  }
}
```

### 5. Promise 静态方法

```javascript
// Promise.all —— 全成功才成功，一个失败立即 reject
MyPromise.all = (promises) => new MyPromise((resolve, reject) => {
  const results = []
  let count = 0
  promises.forEach((p, i) => {
    MyPromise.resolve(p).then(val => {
      results[i] = val
      if (++count === promises.length) resolve(results)
    }, reject)
  })
})

// Promise.race —— 第一个 settled 的接管
MyPromise.race = (promises) => new MyPromise((resolve, reject) => {
  promises.forEach(p => MyPromise.resolve(p).then(resolve, reject))
})
```

## 「其实你每天都在用」

1. **搜索联想**：搜狗输入框的 suggest API，用户每敲一个字母就 debounce 一次。
2. **滚动加载更多**：Scroll 事件 throttle 到每 200ms 检查是否滚到底。
3. **window resize 响应**：resize 事件 throttle，避免布局重算卡死。
4. **按钮防重复提交**：提交按钮 click 后 debounce 或加锁，防止连点多次提交。
5. **`axios.interceptors` 里控制并发**：`Promise.all([req1, req2])` 并行请求，`Promise.race` 做超时兜底。

## 常见误解（FAQ）

**❌ 误区：防抖和节流一个意思。** 防抖是「最后一次说了算」（搜索框）；节流是「按固定频率执行」（滚动、resize）。输入框中你用节流会漏掉最后的输入，滚动中用防抖会等用户停滚才触发——弄反了就功能不对。

**❌ 误区：call/apply/bind 只能绑定 this，没什么用。** `Array.prototype.slice.call(arguments)`、`Math.max.apply(null, arr)`、`Object.prototype.toString.call(x)` 都是经典用法。手写它们不只是面试八股——理解 `fn.call(ctx, ...args)` 背后的 this 切换机制，是搞懂 this 的唯一捷径。

**❌ 误区：bind 返回的函数用 new 调用时，this 还是指向 bind 绑定的对象。** 错。new 绑定的优先级高于显式绑定。如果 `bind` 的实现没有 `this instanceof bound` 的判断，面试直接挂。

**❌ 误区：手写 Promise 必须用 setTimeout 模拟微任务。** 这是过时做法。现代环境用 `queueMicrotask` 或 `Promise.resolve().then()` 来模拟微任务时序，setTimeout 是宏任务，时序不对（宏任务比微任务晚）。

## 一句话总结

手写题不是考你「背没背过」，而是考你是否真的理解了闭包为什么能保留 timer、this 为什么会被改写、Promise 的 then 链为什么能串联——当你写的代码跑通的那一刻，你才真正和内建的 API 站在了同一个理解层面。
