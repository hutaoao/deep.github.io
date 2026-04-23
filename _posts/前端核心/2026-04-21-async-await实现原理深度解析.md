---
title: async/await实现原理深度解析
date: 2026-04-21 10:00:00
author: AI Assistant
categories:
  - 前端核心
  - JavaScript
tags:
  - JavaScript
  - async
  - await
  - Promise
  - 异步编程
---

## 一句话概括

async/await 是 JavaScript 异步编程的语法糖，基于 Generator 函数实现，以同步写法书写异步逻辑，大幅提升代码可读性。

## 背景

在 async/await 出现之前，JavaScript 异步编程主要依靠回调函数和 Promise。回调函数容易陷入「回调地狱」，Promise 虽然解决了回调嵌套问题，但链式调用的语法在复杂逻辑下仍不够直观。async/await 的出现，让开发者可以用同步的写法书写异步代码，兼顾了异步性能和同步可读性。

## 概念与定义

### async 函数

- async 是 ES2017 引入的关键字，用于声明一个异步函数
- async 函数始终返回一个 Promise
- 函数体内可以使用 await 关键字等待 Promise

### await 表达式

- await 只能在 async 函数内部使用
- await 暂停 async 函数的执行，等待 Promise 完成
- Promise resolve 时，await 返回解析值；reject 时抛出错误

## 最小示例

```javascript
// 基本用法
async function fetchData() {
  const response = await fetch('https://api.example.com/data');
  const data = await response.json();
  return data;
}

// 错误处理
async function fetchWithError() {
  try {
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('请求失败:', error);
    return null;
  }
}

// 并发执行
async function fetchAll() {
  const [users, posts] = await Promise.all([
    fetch('/api/users').then(r => r.json()),
    fetch('/api/posts').then(r => r.json())
  ]);
  return { users, posts };
}
```

## 核心知识点拆解

### 1. async 函数的返回值

async 函数返回值会被自动包装为 Promise：

```javascript
async function fn1() { return 1; }           // Promise.resolve(1)
async function fn2() { throw new Error('err'); } // Promise.reject(Error)
async function fn3() { }                      // Promise.resolve(undefined)
```

### 2. await 的执行流程

- await 后表达式立即执行
- 如果是 Promise，直接返回
- 如果不是 Promise，包装为 Promise.resolve(值)
- 暂停函数执行，进入等待
- Promise resolve 后，恢复执行
- Promise reject 后，通过 throw 抛出错误

### 3. 并发控制

```javascript
// 串行执行（错误示例）
async function serial() {
  const a = await fn1(); // 等待
  const b = await fn2(); // 再等待
}

// 并发执行（正确示例）
async function parallel() {
  const [a, b] = await Promise.all([fn1(), fn2()]);
}
```

### 4. 错误传播机制

```javascript
async function test() {
  // try-catch 捕获
  try {
    await promiseThatRejects();
  } catch (e) {
    // 处理错误
  }
}

// Promise.catch 捕获
async function test2() {
  await promiseThatRejects().catch(e => console.error(e));
}
```

## 实战案例

### 案例：实现 retry 重试机制

```javascript
async function retry(fn, maxAttempts = 3, delay = 1000) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      console.log(`尝试 ${attempt} 失败，${delay}ms 后重试...`);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}

// 使用
const data = await retry(() => fetch('/api/data').then(r => r.json()));
```

### 案例：异步队列控制

```javascript
async function asyncQueue(tasks, limit = 3) {
  const results = [];
  const executing = [];
  
  for (const task of tasks) {
    const promise = task().then(result => {
      results.push(result);
    });
    executing.push(promise);
    
    if (executing.length >= limit) {
      await Promise.race(executing);
      const index = executing.findIndex(p => p === promise);
      if (index > -1) executing.splice(index, 1);
    }
  }
  
  await Promise.all(executing);
  return results;
}
```

## 底层原理

### Generator 函数关联

async/await 本质是 Generator 函数的语法糖：

```javascript
// async/await 写法
async function fn() {
  const a = await Promise1();
  const b = await Promise2();
  return a + b;
}

// 等价的 Generator 写法
function* fn() {
  const a = yield Promise1();
  const b = yield Promise2();
  return a + b;
}

// 手动执行器
function run(gen) {
  return new Promise((resolve, reject) => {
    function step(nextFn) {
      let result;
      try {
        result = nextFn();
      } catch (e) {
        return reject(e);
      }
      if (result.done) {
        return resolve(result.value);
      }
      Promise.resolve(result.value).then(
        v => step(() => gen.next(v)),
        e => step(() => gen.throw(e))
      );
    }
    step(() => gen.next());
  });
}
```

### async 函数的编译结果

Babel 编译后的 async 函数：

```javascript
// 源码
async function foo() {
  const a = await 1;
  return a;
}

// 编译后（简化）
function foo() {
  return _asyncToGenerator(function* () {
    const a = yield Promise.resolve(1);
    return a;
  });
}

function _asyncToGenerator(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      const gen = fn.apply(this, args);
      function step(key, arg) {
        let value, error;
        try {
          const info = gen[key](arg);
          value = info.value;
          error = info.done;
        } catch (e) {
          reject(e);
          return;
        }
        if (error) {
          resolve(value);
        } else {
          Promise.resolve(value).then(v => step('next', v), e => step('throw', e));
        }
      }
      step('next');
    });
  };
}
```

### 微任务队列

await 后面的 Promise 会加入微任务队列：

```javascript
async function demo() {
  console.log(1);
  await Promise.resolve();
  console.log(2); // 下一轮微任务
}
console.log(3);
demo();
console.log(4);
// 输出: 3, 1, 4, 2
```

## 高频面试题解析

### Q1: async/await 和 Promise 的关系？

async/await 是 Promise 的语法糖，底层仍使用 Promise。每个 await 对应一个 .then()，async 函数的返回值自动包装为 Promise。

### Q2: await 后的代码一定在 Promise resolve 后执行吗？

不一定。如果 await 后不是 Promise，会立即包装为 Promise.resolve()。但无论何种情况，await 后的代码都会作为微任务执行。

### Q3: 如何并行执行多个 async 函数？

使用 Promise.all() 包裹多个 async 函数调用：

```javascript
const [a, b] = await Promise.all([async1(), async2()]);
```

### Q4: async 函数中的 throw 相当于什么？

相当于 Promise.reject()：

```javascript
async function fn() {
  throw new Error('err');
}
// 等价于
function fn() {
  return Promise.reject(new Error('err'));
}
```

### Q5: 下面代码的输出顺序？

```javascript
async function async1() {
  console.log('1');
  await async2();
  console.log('2');
}
async function async2() {
  console.log('3');
}
console.log('4');
async1();
console.log('5');
```

答案：4, 1, 3, 5, 2

解释：async2() 同步执行完（输出 3），await 后面的 console.log(2) 加入微任务队列，在同步代码结束后执行。

## 总结与扩展

### 核心要点

1. async 函数始终返回 Promise
2. await 暂停函数执行，等待 Promise 完成
3. await 后的代码作为微任务执行
4. 错误通过 try-catch 或 Promise.catch 捕获
5. 使用 Promise.all 实现并发执行

### 扩展阅读

- ES2017 规范定义：https://tc39.es/ecma262/#sec-async-function-definitions
- 深入理解 Event Loop 与微任务
- 探索 async 并发控制库如 p-limit 的实现

### 相关手写题

- 手写 Promise 类
- 手写 asyncToGenerator
- 实现带并发限制的请求调度器