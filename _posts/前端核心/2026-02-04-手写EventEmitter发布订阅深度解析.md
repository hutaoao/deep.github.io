---
title: "手写EventEmitter发布订阅深度解析"
date: 2026-02-04
categories: [前端核心, JavaScript, 设计模式, 发布订阅]
tags: [EventEmitter, 发布订阅模式, 事件驱动, 手写题]
---

## 📌 一句话概括

EventEmitter是Node.js核心模块，实现了发布订阅模式，掌握其手写实现能帮你深入理解事件驱动编程。

## 🎯 背景

在Node.js中，很多核心API都基于事件驱动，比如 `http.Server`、`fs.ReadStream` 等。它们都继承自 `EventEmitter`，通过 `on` 监听事件，通过 `emit` 触发事件。

**为什么要手写EventEmitter？**
1. **面试高频题：** 考察对设计模式的理解
2. **理解Node.js核心：** EventEmitter是Node.js事件驱动的基础
3. **实际应用场景：** 自定义事件系统、组件通信等

## 💡 概念与定义

### 什么是发布订阅模式

**发布订阅模式（Pub/Sub Pattern）** 是一种消息传递模式，发送者（发布者）不会直接发送消息给接收者（订阅者），而是通过事件中心进行通信。

**核心角色：**
1. **发布者（Publisher）：** 触发事件
2. **订阅者（Subscriber）：** 监听事件
3. **事件中心（Event Channel）：** 管理事件与回调

### EventEmitter核心API

| 方法 | 功能 | 说明 |
|------|------|------|
| `on(event, callback)` | 监听事件 | 可多次触发 |
| `emit(event, ...args)` | 触发事件 | 执行所有回调 |
| `off(event, callback)` | 取消监听 | 移除指定回调 |
| `once(event, callback)` | 单次监听 | 触发一次后自动移除 |

## 🔍 核心知识点拆解

### 1. 基础EventEmitter实现

**核心思路：** 用对象存储事件名和回调函数数组。

```javascript
class EventEmitter {
  constructor() {
    this.events = {};  // 存储事件和回调
  }

  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);
    return this;  // 支持链式调用
  }

  emit(event, ...args) {
    const callbacks = this.events[event] || [];
    callbacks.forEach(callback => callback(...args));
    return this;
  }

  off(event, callback) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(cb => cb !== callback);
    }
    return this;
  }

  once(event, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(event, wrapper);  // 执行后移除自己
    };
    this.on(event, wrapper);
    return this;
  }
}
```

**使用示例：**

```javascript
const emitter = new EventEmitter();

// 监听事件
emitter.on('message', (msg) => {
  console.log('收到消息：', msg);
});

// 触发事件
emitter.emit('message', 'Hello World!');
// 输出：收到消息：Hello World!
```

### 2. 处理once的坑

**问题：** 如果直接用 `once` 包装后的函数引用，用户无法手动 `off` 移除监听器。

**解决：** 在原始回调上存储包装函数引用。

```javascript
class EventEmitter {
  // ... 其他代码

  once(event, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(event, wrapper);
    };
    // 在原始回调上存储wrapper引用，方便移除
    callback._wrapper = wrapper;
    this.on(event, wrapper);
    return this;
  }

  off(event, callback) {
    if (this.events[event]) {
      const index = this.events[event].indexOf(callback._wrapper || callback);
      if (index > -1) {
        this.events[event].splice(index, 1);
      }
    }
    return this;
  }
}
```

### 3. 支持最大监听器限制

**背景：** Node.js的EventEmitter默认最多10个监听器，防止内存泄漏。

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
    this.maxListeners = 10;  // 默认最大监听器数
  }

  setMaxListeners(n) {
    this.maxListeners = n;
  }

  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }

    // 检查监听器数量
    if (this.events[event].length >= this.maxListeners) {
      console.warn(`MaxListenersExceededWarning: ${event}`);
    }

    this.events[event].push(callback);
    return this;
  }

  // ... 其他方法
}
```

### 4. 支持事件前缀（如newListener、removeListener）

**背景：** Node.js的EventEmitter在添加/移除监听器时会触发特殊事件。

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
    this.maxListeners = 10;
  }

  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }

    // 触发newListener事件
    if (event !== 'newListener' && this.events['newListener']) {
      this.events['newListener'].forEach(cb => cb(event, callback));
    }

    if (this.events[event].length >= this.maxListeners) {
      console.warn(`MaxListenersExceededWarning: ${event}`);
    }

    this.events[event].push(callback);
    return this;
  }

  off(event, callback) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(cb => cb !== callback);

      // 触发removeListener事件
      if (event !== 'removeListener' && this.events['removeListener']) {
        this.events['removeListener'].forEach(cb => cb(event, callback));
      }
    }
    return this;
  }

  // ... 其他方法
}
```

## 🛠️ 实战案例

### 案例1：Node.js中的EventEmitter实际使用

```javascript
const EventEmitter = require('events');

class MyServer extends EventEmitter {
  constructor() {
    super();
    this.clients = 0;
  }

  connect(client) {
    this.clients++;
    this.emit('connect', client);
  }

  disconnect(client) {
    this.clients--;
    this.emit('disconnect', client);
  }
}

const server = new MyServer();

server.on('connect', (client) => {
  console.log(`客户端 ${client} 连接，当前连接数：${server.clients}`);
});

server.on('disconnect', (client) => {
  console.log(`客户端 ${client} 断开，当前连接数：${server.clients}`);
});

server.connect('Alice');
server.connect('Bob');
server.disconnect('Alice');
```

### 案例2：前端组件通信

```javascript
// eventBus.js
class EventBus extends EventEmitter {}

export default new EventBus();

// ComponentA.js
import eventBus from './eventBus';

eventBus.on('data-update', (data) => {
  console.log('ComponentA收到数据更新：', data);
});

// ComponentB.js
import eventBus from './eventBus';

eventBus.emit('data-update', { id: 1, name: 'Alice' });
```

### 案例3：Promise并发控制（结合EventEmitter）

```javascript
class PromisePool extends EventEmitter {
  constructor(tasks, maxConcurrency) {
    super();
    this.tasks = tasks;
    this.maxConcurrency = maxConcurrency;
    this.running = 0;
    this.index = 0;
    this.results = [];
  }

  async run() {
    return new Promise((resolve) => {
      const next = async () => {
        if (this.index >= this.tasks.length && this.running === 0) {
          this.emit('finish', this.results);
          resolve(this.results);
          return;
        }

        while (this.running < this.maxConcurrency && this.index < this.tasks.length) {
          const taskIndex = this.index++;
          this.running++;

          this.tasks[taskIndex]().then(result => {
            this.results[taskIndex] = result;
            this.running--;
            this.emit('task-complete', { index: taskIndex, result });
            next();
          });
        }
      };

      next();
    });
  }
}
```

## 📐 底层原理

### Node.js的EventEmitter实现

**核心数据结构：**

```c
// Node.js源码（简化版）
class EventEmitter {
  constructor() {
    this._events = Object.create(null);  // 使用null原型，避免原型链污染
  }
}
```

**性能优化技巧：**

1. **单次监听优化：** 如果某个事件只有一个监听器，直接存储函数而不是数组
   ```javascript
   on(event, callback) {
     if (!this._events[event]) {
       this._events[event] = callback;  // 直接存储函数
     } else if (typeof this._events[event] === 'function') {
       this._events[event] = [this._events[event], callback];  // 转为数组
     } else {
       this._events[event].push(callback);
     }
   }
   ```

2. **移除监听器优化：** 如果只有一个监听器，直接删除属性而不是过滤数组
   ```javascript
   off(event, callback) {
     const listeners = this._events[event];
     if (typeof listeners === 'function') {
       if (listeners === callback) {
         delete this._events[event];
       }
     } else if (Array.isArray(listeners)) {
       // ... 过滤数组
     }
   }
   ```

### 发布订阅模式 vs 观察者模式

| 特性 | 发布订阅模式 | 观察者模式 |
|------|-------------|-----------|
| 耦合度 | 松耦合（通过事件中心） | 紧耦合（直接依赖） |
| 通信方式 | 异步（事件驱动） | 同步（直接调用） |
| 典型实现 | EventEmitter | Subject/Observer |

**观察者模式示例：**

```javascript
class Subject {
  constructor() {
    this.observers = [];
  }

  addObserver(observer) {
    this.observers.push(observer);
  }

  notify(data) {
    this.observers.forEach(observer => observer.update(data));
  }
}

class Observer {
  update(data) {
    console.log('收到更新：', data);
  }
}
```

## 🎓 高频面试题解析

### Q1: 手写EventEmitter的核心思路是什么？

**答：**
1. 用对象存储事件名和回调函数数组
2. `on`：将回调加入数组
3. `emit`：遍历数组执行所有回调
4. `off`：过滤掉指定回调
5. `once`：执行一次后自动移除

### Q2: 如何处理once的坑（用户无法手动off）？

**答：**
在原始回调上存储包装函数引用：
```javascript
once(event, callback) {
  const wrapper = (...args) => {
    callback(...args);
    this.off(event, wrapper);
  };
  callback._wrapper = wrapper;  // 存储引用
  this.on(event, wrapper);
}
```

### Q3: EventEmitter和发布订阅模式的关系？

**答：**
EventEmitter是发布订阅模式在Node.js中的实现。发布订阅模式是设计模式，EventEmitter是具体实现。

### Q4: 如何防止内存泄漏？

**答：**
1. 设置 `maxListeners` 限制
2. 及时调用 `off` 移除不需要的监听器
3. 使用 `once` 代替 `on`（自动移除）

## 📝 总结与扩展

### 核心要点

1. **EventEmitter核心：** 用对象存储事件和回调
2. **四大API：** `on`、`emit`、`off`、`once`
3. **关键优化：** 最大监听器限制、单次监听优化
4. **实际应用：** Node.js核心API、前端组件通信

### 扩展方向

1. **结合Promise：** 实现事件转Promise
2. **结合RxJS：** 将EventEmitter转为Observable
3. **跨进程通信：** 将EventEmitter扩展到多进程（如Electron）

### 完整代码

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
    this.maxListeners = 10;
  }

  setMaxListeners(n) { this.maxListeners = n; }

  on(event, callback) {
    if (!this.events[event]) this.events[event] = [];
    if (this.events[event].length >= this.maxListeners) {
      console.warn(`MaxListenersExceededWarning: ${event}`);
    }
    this.events[event].push(callback);
    return this;
  }

  emit(event, ...args) {
    const callbacks = this.events[event] || [];
    callbacks.forEach(callback => callback(...args));
    return this;
  }

  off(event, callback) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(cb => cb !== callback);
    }
    return this;
  }

  once(event, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(event, wrapper);
    };
    callback._wrapper = wrapper;
    this.on(event, wrapper);
    return this;
  }
}
```

### 相关资源

- Node.js官方文档 - events模块
- 《JavaScript设计模式》- 发布订阅模式
- RxJS官方文档 - Observable

---

**本文从基础实现到高级优化，全面解析了EventEmitter的手写实现，帮助你掌握发布订阅模式的核心原理。**
