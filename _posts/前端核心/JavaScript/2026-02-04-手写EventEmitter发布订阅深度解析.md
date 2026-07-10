---
layout: post
title: "手写 EventEmitter"
date: 2026-02-04
categories: ["前端核心", "JavaScript"]
tags: ["EventEmitter", "发布订阅", "设计模式", "手写", "面试"]
---

## 一句话概括

EventEmitter 是发布订阅模式的标准实现——用一个对象存事件名到回调数组的映射，`on` 订阅、`emit` 触发、`off` 移除、`once` 自动化清理。面试考的是你处理 `once` 的包装技巧和 `emit` 中途移除回调的边界情况。

## 核心知识点

### 1. 骨架 —— 事件名 → 回调数组

```javascript
class EventEmitter {
  constructor() {
    this._events = Object.create(null); // { 'click': [fn1, fn2] }
  }

  on(event, fn) {
    (this._events[event] ??= []).push(fn);
    return this; // 支持链式调用
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

就这么简单。但面试官会让你在此基础上暴露盲点。

### 2. once —— 执行一次后自毁

```javascript
once(event, fn) {
  const wrapper = (...args) => {
    this.off(event, wrapper);
    fn.apply(this, args);
  };
  wrapper._original = fn; // 保存原始引用，方便外部 off
  return this.on(event, wrapper);
}

// off 升级：能移除 once 的回调
off(event, fn) {
  const fns = this._events[event];
  if (!fns) return this;
  this._events[event] = fns.filter(cb => cb !== fn && cb._original !== fn);
  return this;
}
```

**关键技巧：** `wrapper._original = fn` 让用户能通过原始函数引用来 `off` 一个 `once` 注册的回调。

### 3. 监听器数量限制 —— 内存泄漏预警

```javascript
constructor() {
  this._events = Object.create(null);
  this._maxListeners = 10; // Node.js 默认值
}

on(event, fn) {
  const fns = (this._events[event] ??= []);
  fns.push(fn);
  if (fns.length > this._maxListeners) {
    console.warn(
      `MaxListenersExceededWarning: Possible memory leak. ` +
      `${fns.length} listeners on [${event}]`
    );
  }
  return this;
}
```

这个警告是内存泄漏的信号——如果你不断 `on` 但从不清掉（比如组件销毁后没 `off`），回调会越堆越多。

### 4. 性能优化 —— 单监听器不走数组

```javascript
emit(event, ...args) {
  const handler = this._events[event];
  if (!handler) return false;

  if (typeof handler === 'function') {
    // 只有一个监听器时，handler 是函数不是数组（Node.js 实际实现）
    handler.apply(this, args);
  } else {
    // slice 一份再遍历，避免中途 off 导致索引错乱
    for (const fn of handler.slice()) {
      fn.apply(this, args);
    }
  }
  return true;
}
```

**为什么 `handler.slice()`？** 如果 `emit` 过程中某个回调 `off` 了自己，直接 `forEach` 会导致后续回调被跳过。Slice 一份再遍历，原数组被修改也不影响当前 emit。

### 5. 发布订阅 vs 观察者模式

```javascript
// 发布订阅：有中间层（EventEmitter），双方互不知道对方存在
emitter.on('data', callback);  // 订阅者
emitter.emit('data', payload);  // 发布者，不知道谁在听

// 观察者模式：Subject 直接持有 Observer 列表
subject.addObserver(observer);  // 紧耦合
subject.notify();               // Subject 知道每个 Observer 的 update 接口
```

**面试结论：** 发布订阅多一层事件通道，耦合更松；观察者直接相连，交互更明确。Vue 3 响应式是观察者，EventEmitter 是发布订阅。

## 其实你每天都在用

- **Node.js `http.createServer()`** — server 对象继承自 EventEmitter，`server.on('request', handler)` 就是订阅
- **Vue 2 的 `$on / $emit / $once`** — 组件事件总线是 EventEmitter 的完美复刻
- **WebSocket `ws.on('message', ...)`** — 事件驱动通信的标准范式
- **`process.on('uncaughtException', ...)`** — Node 全局异常捕获本质就是事件订阅
- **组件通信的 EventBus** — 在没有 Pinia/Redux 的小项目里，一个 EventEmitter 实例搞定跨组件通信

## 常见误解

- **❌ 误区：「emit 是异步的」** emit 是**同步**遍历回调数组逐一执行的。之所以叫"事件驱动"而不是"异步"，是个常见的认知偏差。如果在回调里做耗时操作，会阻塞后续回调。

- **❌ 误区：「emit 过程中 off 自己完全安全」** 不安全。如果 `emit` 直接对原数组 `forEach`，中途 `off` 自己后数组索引错乱，后面的回调被跳过。解决方案是遍历前 `slice()` 一份。

- **❌ 误区：「maxListeners 设大就行，警告无所谓」** 这个警告是告诉你"可能有内存泄漏"。组件被销毁后忘了 `off`，事件的回调数组会无限增长——这才是危险，限制只是预警。

- **❌ 误区：「EventEmitter 就是观察者模式」** 不完全对。观察者模式中 Subject 直接知道 Observer 是谁；发布订阅通过事件通道解耦，双方互不认识。一个是直连，一个是有中间层。

## 一句话总结

一个 Map 存事件、一个数组存回调、四个方法 CRUD——EventEmitter 本体极简，复杂度全在边界：`once` 的 wrapper 代理、`slice` 的遍历安全、`_maxListeners` 的内存预警。三关全过，面试稳了。
