---
layout: post
title: "手写 EventEmitter 发布订阅"
date: 2026-02-04
categories: ["前端核心", "JavaScript"]
tags: ["EventEmitter", "发布订阅", "手写题", "设计模式"]
---

## 一句话概括

EventEmitter 是发布订阅模式在 Node.js 中的实现——用 `on` 订阅、`emit` 发布、`off` 取消、`once` 只执行一次，底层就是用一个对象存事件名到回调数组的映射。

## 核心知识点

### 1. 基础骨架 — 事件名 → 回调数组

```javascript
class EventEmitter {
  constructor() {
    this._events = {};  // { 'click': [fn1, fn2], 'load': [fn3] }
  }

  on(event, fn) {
    (this._events[event] ??= []).push(fn);
    return this;  // 支持链式调用
  }

  emit(event, ...args) {
    this._events[event]?.forEach(fn => fn(...args));
    return this;
  }

  off(event, fn) {
    this._events[event] = this._events[event]?.filter(cb => cb !== fn);
    return this;
  }
}
```

就这么简单。但面试不会只让你写到这。

### 2. once — 执行一次后自动移除

```javascript
once(event, fn) {
  const wrapper = (...args) => {
    fn(...args);
    this.off(event, wrapper); // 执行后立刻移除
  };
  // 关键：把 wrapper 挂到 fn 上，方便用户手动 off
  wrapper._original = fn;
  this.on(event, wrapper);
  return this;
}

// off 也要升级——允许通过原始 fn 找到 wrapper 并移除
off(event, fn) {
  const fns = this._events[event];
  if (!fns) return this;
  // fn 可能是 once 的原始回调，也可能是普通回调
  this._events[event] = fns.filter(cb => cb !== fn && cb._original !== fn);
  return this;
}
```

### 3. 监听器数量限制 — 防止内存泄漏

```javascript
constructor() {
  this._events = {};
  this._maxListeners = 10; // Node.js 默认值
}

on(event, fn) {
  const fns = (this._events[event] ??= []);
  fns.push(fn);
  // 超过阈值打 warn
  if (fns.length > this._maxListeners) {
    console.warn(
      `MaxListenersExceededWarning: Possible EventEmitter memory leak ` +
      `detected. ${fns.length} listeners added to [${event}].`
    );
  }
  return this;
}

setMaxListeners(n) {
  this._maxListeners = n;
  return this;
}
```

### 4. 性能优化 — 单监听器时用函数而非数组

```javascript
emit(event, ...args) {
  const handler = this._events[event];
  if (!handler) return false;
  if (typeof handler === 'function') {
    handler(...args);   // 单监听器，直接调
  } else {
    handler.slice().forEach(fn => fn(...args)); // slice 避免中途 off 影响遍历
  }
  return true;
}
```

**这是 Node.js EventEmitter 的实际实现细节。** 大多数事件只有一个监听器，用数组存太浪费。

### 5. 发布订阅 vs 观察者模式

```javascript
// 发布订阅：通过事件中心解耦
emitter.on('data', subscriber);
emitter.emit('data'); // 发布者不知道订阅者是谁

// 观察者模式：Subject 直接持有 Observer 列表
subject.addObserver(observer);
subject.notify(); // 紧耦合，Subject 知道 Observer 接口
```

**面试结论：** 发布订阅有中间层（EventEmitter），耦合更松；观察者模式是直连的。

## 其实你每天都在用

- **Node.js 的 `http.createServer()`** 返回的对象就是 EventEmitter，`server.on('request', ...)` 就是订阅
- **Vue 2 的 `$on` / `$emit` / `$once`** — 组件事件总线是 EventEmitter 的完美复刻
- **WebSocket 的 `ws.on('message', ...)`** — 事件驱动的核心范式
- **`process.on('uncaughtException', ...)`** — Node.js 全局异常捕获本质上就是事件订阅
- **自定义事件总线解决跨组件通信** — 在没有 Vuex/Pinia 的小项目里，一个 EventEmitter 实例就能搞定

## 常见误解

- **❌ 误区：「emit 触发后回调是同步执行的」** 对。`emit` 是同步遍历回调数组逐一执行，不是 `setTimeout`。如果把耗时操作放在回调里，会阻塞后续回调。之所以叫「事件驱动」而不是「异步」，这是一个常见的概念混淆。

- **❌ 误区：「off 在 forEach 过程中移除元素安全」** 不安全。如果你在 `emit` 的 forEach 循环里 off 掉自己，数组在遍历中被修改，后面的回调可能被跳过。Node.js 的解决方案是 `handler.slice().forEach(...)` 复制一份再遍历。

- **❌ 误区：「EventEmitter 就是观察者模式」** 不完全对。发布订阅模式和观察者模式的核心区别是**中间层**。观察者模式中 Subject 直接知道 Observer 是谁；发布订阅模式中双方通过 Event Channel 通信，互相不认识。

- **❌ 误区：「maxListeners=10 设大了就行，不用管」** 这个警告是**内存泄漏的信号**。如果你的 `on` 不断往同一个事件上挂回调但从不清掉——比如组件被销毁却没 `off`——这个事件会越攒越大。限值只是一个提醒，不是解决方案。

## 一句话总结

一个对象存事件→一个数组存回调→四个方法 CRUD——EventEmitter 的复杂度全在边界处理，掌握了 `once` 的 wrapper 技巧和 `slice` 的遍历安全，面试稳了。
