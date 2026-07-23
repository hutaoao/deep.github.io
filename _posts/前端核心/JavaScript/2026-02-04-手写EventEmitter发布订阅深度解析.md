---
layout: post
title: "手写 EventEmitter"
date: 2026-02-04
categories: ["前端核心", "JavaScript"]
tags: ["EventEmitter", "发布订阅", "设计模式", "手写", "面试"]
---

## 一句话概括

EventEmitter 是发布订阅模式的教科书级实现：`on` 存回调、`emit` 逐个调、`off` 精准删、`once` 一次自毁。面试不考这四个 API 的调用——考的是 `once` 的 wrapper 代理技巧、`emit` 中 `off` 自己的遍历安全、和观察者模式的本质区别。

## 核心知识点

### 1. 骨架——事件到回调数组的映射

```js
class EventEmitter {
  constructor() {
    this._events = Object.create(null); // {'click': [fn1, fn2], 'load': [fn3]}
  }

  on(event, fn) {
    (this._events[event] ??= []).push(fn);
    return this; // 链式调用
  }

  emit(event, ...args) {
    this._events[event]?.forEach(fn => fn(...args));
    return this;
  }

  off(event, fn) {
    const fns = this._events[event];
    if (fns) this._events[event] = fns.filter(cb => cb !== fn);
    return this;
  }
}
```

这 20 行能跑——但藏着两个面试要考的坑，往下看。

### 2. once——wrapper 代理模式

```js
once(event, fn) {
  const wrapper = (...args) => {
    this.off(event, wrapper);   // 先解绑，再执行（防止 fn 里再次 emit 导致重入）
    fn.apply(this, args);
  };
  wrapper._original = fn;      // 🔑 保存原始引用——用户能用原始函数来 off
  return this.on(event, wrapper);
}

// off 升级：同时匹配直接 on 的和 once 包装的
off(event, fn) {
  const fns = this._events[event];
  if (!fns) return this;
  this._events[event] = fns.filter(cb => cb !== fn && cb._original !== fn);
  return this;
}
```

**核心技巧：** `wrapper._original = fn` 建立包装函数到原始函数的桥——`emitter.off('click', fn)` 不管 `fn` 是 `on` 的还是 `once` 的，都能精准命中。

### 3. 遍历安全——`slice()` 保命

```js
emit(event, ...args) {
  const fns = this._events[event];
  if (!fns) return false;

  // 🔑 slice() 拷贝一份——原数组被 callback 修改不影响本次 emit
  for (const fn of fns.slice()) {
    fn.apply(this, args);
  }
  return true;
}
```

**坑在哪：** 回调 A 里调了 `off('event', B)` → 回调数组变成 `[A, C]` → 遍历器在原数组上继续走 → 索引 1 现在是 C 但遍历器以为到了 B → C 被**静默跳过**。

`slice()` 拷贝一份 → 原数组随便被 callback 改，遍历器永远走的是那一刻的快照。

### 4. 最大监听数——内存泄漏预警

```js
constructor() {
  this._events = Object.create(null);
  this._maxListeners = 10;
}

on(event, fn) {
  const fns = (this._events[event] ??= []);
  fns.push(fn);
  if (fns.length > this._maxListeners) {
    console.warn(`⚠️ [${event}] has ${fns.length} listeners — possible memory leak`);
  }
  return this;
}
```

警告是信号：组件卸载没 `off` → 回调数组无限增长 → 闭包引用的 DOM / 数据全被 EventEmitter 死抓着不放。

### 5. 发布订阅 vs 观察者——面试必问

```js
// 发布订阅：中间有事件通道，双方互不认识
bus.on('data', cb);     // 订阅
bus.emit('data', val);  // 发布——不知道谁在听

// 观察者：Subject 直接持有 Observer 列表
subject.addObserver(obs);
subject.notify();       // subject 直接调 obs.update()

// 差异：
// 发布订阅 = 松耦合（通过事件名交互），代表：EventEmitter、EventBus
// 观察者   = 紧耦合（Subject 认识每个 Observer），代表：Vue 响应式、MutationObserver
```

## 其实你每天都在用

- **Node.js `http.createServer()`**：server 继承 EventEmitter，`server.on('request', handler)` 就是发布订阅
- **Vue 2 `$on / $emit / $once`**：EventEmitter 直接复刻，Vue 3 弃用但在 mitt 里重生
- **WebSocket `ws.on('message', cb)`**：事件驱动通信的标准范式
- **`process.on('uncaughtException')`**：Node 全局异常捕获就是事件订阅
- **微前端通信**：跨子应用的全局事件总线底层就是 EventEmitter

## 常见误解

- **❌ 误区：「emit 是异步的」** emit **完全同步**——遍历回调逐个立即执行。回调里做耗时操作会阻塞后续回调。`process.nextTick` 或 `Promise.resolve()` 不在 emit 的实现里。

- **❌ 误区：「emit 里 off 自己不碍事」** 直接对原数组遍历时 `off` 自己 → 数组长度变了 → 索引错乱 → 后续回调静默跳过。解法：`fns.slice()` 快照遍历。

- **❌ 误区：「maxListeners 设大就行」** 掩耳盗铃。警告告诉你"有内存泄漏风险"——组件销毁忘 `off`，回调连带着闭包数据全被死抓着。治本：`unmount` 时清掉所有监听。

- **❌ 误区：「EventEmitter 就是观察者模式」** 差一个中间层。观察者的 Subject 直接知道谁在 watch 它；发布订阅的双方通过事件名交互，不直接持有对方引用。前者紧耦合交互明确，后者松耦合灵活。

## 一句话总结

一个 Map 存事件、一个数组存回调、四个方法 CRUD——EventEmitter 就这么简单。真正考的是三个边界：`once` 的 wrapper 代理、`slice()` 的遍历防御、`_maxListeners` 的预警意识。三关全过，面试官直接下一题。
