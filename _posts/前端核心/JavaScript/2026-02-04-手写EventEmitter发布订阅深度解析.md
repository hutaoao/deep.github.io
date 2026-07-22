---
layout: post
title: "手写 EventEmitter"
date: 2026-02-04
categories: ["前端核心", "JavaScript"]
tags: ["EventEmitter", "发布订阅", "设计模式", "手写", "面试"]
---

## 一句话概括

EventEmitter 是发布订阅模式的教科书实现：`on` 存回调、`emit` 逐个调、`off` 精准删、`once` 一次后自毁。面试的核心不是这四个方法的调用，而是 `once` 的 wrapper 代理技巧和 `emit` 过程中 `off` 自己的遍历安全问题。

## 核心知识点

### 1. 骨架 — 事件名到回调数组的映射

```javascript
class EventEmitter {
  constructor() {
    this._events = Object.create(null);  // { 'click': [fn1, fn2], 'resize': [fn3] }
  }

  on(event, fn) {
    (this._events[event] ??= []).push(fn);
    return this;  // 支持链式调用: emitter.on('a', fn1).on('b', fn2)
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

  removeAllListeners(event) {
    delete this._events[event];
    return this;
  }
}

// 就这么短——但它的 bug 藏在下文
```

### 2. once — wrapper 代理模式

```javascript
once(event, fn) {
  const wrapper = (...args) => {
    this.off(event, wrapper);  // 先解绑，再执行（防止 fn 里再次 emit 同一事件导致 wrapper 被调两次）
    fn.apply(this, args);
  };
  wrapper._original = fn;     // 🔑 保存原始引用，用户能用原始函数来 off
  return this.on(event, wrapper);
}

// off 升级版：能识别 once 注册的回调
off(event, fn) {
  const fns = this._events[event];
  if (!fns) return this;
  this._events[event] = fns.filter(cb => cb !== fn && cb._original !== fn);
  return this;
}
```

**核心技巧：** `wrapper._original = fn` 让包装函数和原始函数之间建立了双向桥——用户 `emitter.off('click', handleClick)` 能同时命中直接 `on` 的和 `once` 注册的同名回调。

### 3. 遍历安全 — `slice()` 防止中途 off

```javascript
emit(event, ...args) {
  const handler = this._events[event];
  if (!handler) return false;

  if (typeof handler === 'function') {
    handler.apply(this, args);                // 只有一个监听器：直接执行
  } else {
    for (const fn of handler.slice()) {       // 🔑 slice() 是关键
      fn.apply(this, args);
    }
  }
  return true;
}
```

**为什么必须 `slice()`？** 如果 `emit` 过程中某个回调 `off` 了自己，直接对原数组 `forEach` 会导致索引错乱——`[fnA, fnB, fnC]` 遍历到 fnB 时被删除，fnC 的索引从 2 变成 1，但遍历器已经往下走了，fnC 被跳过。`slice()` 拷贝一份，原数组的修改不影响当前 emit。

### 4. 最大监听数 — 内存泄漏预警

```javascript
constructor() {
  this._events = Object.create(null);
  this._maxListeners = 10;      // Node.js EventEmitter 的默认值
}

on(event, fn) {
  const fns = (this._events[event] ??= []);
  fns.push(fn);
  if (fns.length > this._maxListeners) {
    console.warn(
      `MaxListenersExceededWarning: Possible memory leak detected. ` +
      `${fns.length} listeners on [${event}] event.`
    );
  }
  return this;
}
```

这个警告是信号：如果组件卸载后没 `off`，事件的回调数组无限增长——内存泄漏就这么来的。限制不是目的，警告才是。

### 5. 发布订阅 vs 观察者模式 — 面试必问

```javascript
// 发布订阅：有中间层，双方互不知道对方
eventBus.on('data', callback);   // 订阅者
eventBus.emit('data', payload);  // 发布者，不知道谁在听

// 观察者模式：Subject 直接持有 Observer 列表，紧耦合
subject.addObserver(observer);
subject.notify();  // Subject 直接调 observer.update()
```

| | 发布订阅 | 观察者模式 |
|---|---|---|
| 耦合度 | 松（通过事件通道） | 紧（Subject 认识 Observer） |
| 中间层 | ✅ 事件调度中心 | ❌ 直接连接 |
| 代表 | EventEmitter, EventBus | Vue 3 响应式系统, MutationObserver |

## 其实你每天都在用

- **Node.js `http.createServer()`**：server 继承自 EventEmitter，`server.on('request', handler)` 就是经典的发布订阅
- **Vue 2 组件事件总线**：`$on / $emit / $once` 是 EventEmitter 的直接复刻
- **WebSocket `ws.on('message', cb)`**：事件驱动通信的标准范式
- **`process.on('uncaughtException', fn)`**：Node 全局异常捕获就是事件订阅
- **微前端通信**：用 EventEmitter 做跨子应用的全局事件总线

## 常见误解

- **❌ 误区：「emit 是异步的」** emit **完全同步**——逐个遍历回调数组立即执行。`setTimeout` 或 `Promise.resolve()` 不在 emit 的实现里。回调里做耗时操作会阻塞后续回调。

- **❌ 误区：「emit 中 off 自己不碍事」** 对原数组直接遍历时 `off` 自己 → 数组长度变了 → 索引错乱 → 后续回调被跳过。解决方案：`handler.slice()` 拷贝一份再遍历。

- **❌ 误区：「maxListeners 设大点就行」** 这是掩耳盗铃。警告的意义是告诉你"有内存泄漏风险"——组件销毁后忘了 `off`，回调对象和闭包里引用的 DOM/fetched 数据全被 EventEmitter 死抓着不放。治本：在组件 `unmount` 时清掉所有监听。

- **❌ 误区：「EventEmitter 就是观察者模式」** 差一个中间层。观察者模式的 Subject 直接知道谁在 watch 它；发布订阅的双方通过事件名交互，不直接持有对方引用。前者紧耦合、交互明确；后者松耦合、灵活性高。

## 一句话总结

一个 Map 存事件、一个数组存回调、四个方法 CRUD——EventEmitter 本体就这么简单。真正考的是三个边界：`once` 的 wrapper 代理、`slice()` 的遍历防御、`_maxListeners` 的预警意识。三关全过，面试官直接下一题。
