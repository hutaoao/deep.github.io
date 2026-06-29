---
title: "async-await实现原理深度解析"
date: 2026-01-12 08:00:00 +0800
categories: [前端核心, JavaScript]
tags: [JavaScript, async, await, Promise, 异步编程, Generator]
description: "揭开 async/await 的语法糖面纱——底层是 Generator + Promise 自动执行器，await 等价于 yield + then。"
---

## 一句话概括

`async/await` 不是新魔法，是 **Generator + Promise 自动执行器**的语法糖——把 `yield` 换成 `await`，把手动 `.next()` 换成引擎自动推进，让你用同步方式写异步代码。

---

## 核心知识点

### 1. async 函数的返回值永远是 Promise

```js
async function fn1() { return 42; }
fn1(); // Promise { 42 }——自动包装

async function fn2() { throw new Error('boom'); }
fn2(); // Promise { <rejected> Error: boom }
```

规则简单到离谱：return 普通值 → `Promise.resolve` 包装；throw → `Promise.reject` 包装；什么都不 return → `Promise.resolve(undefined)`。

### 2. await 暂停的是 async 函数，不是主线程

```js
console.log('1');
async function demo() {
  console.log('2');
  await Promise.resolve();
  console.log('3'); // 微任务
}
demo();
console.log('4');
// 输出：1 → 2 → 4 → 3
```

`await` 之后的代码等价于 `.then(() => { ... })`——被推入微任务队列，在同步代码跑完后执行。

### 3. Generator 版 async/await（关键！）

```js
function* myAsync() {
  const a = yield Promise.resolve(1);
  const b = yield Promise.resolve(a + 2);
  return b;
}

// 自动执行器
function run(gen) {
  return new Promise((resolve, reject) => {
    const g = gen();
    function step(next) {
      let result;
      try { result = g[next](); } catch (e) { return reject(e); }
      if (result.done) return resolve(result.value);
      Promise.resolve(result.value).then(
        v => step('next'),
        e => g.throw(e)
      );
    }
    step('next');
  });
}

run(myAsync).then(console.log); // 3
```

这就是 `async/await` 的真相：**Generator 负责暂停/恢复，Promise 负责异步，自动执行器负责推动**。

### 4. 错误处理两种姿势

```js
// 姿势一：try/catch（推荐）
async function fn() {
  try {
    const data = await fetchData();
  } catch (e) {
    console.error('请求失败', e);
  }
}

// 姿势二：.catch()
async function fn2() {
  const data = await fetchData().catch(e => null);
}
```

`try/catch` 能抓 `await` 后面的 rejected Promise，因为 reject 被转换为 Generator 的 `throw`。

### 5. 并发 vs 串行 —— 性能差一个数量级

```js
// ❌ 串行：总耗时 = t1 + t2 + t3
const a = await fetchA();
const b = await fetchB();
const c = await fetchC();

// ✅ 并发：总耗时 ≈ max(t1, t2, t3)
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);
```

`await` 之间没有依赖就用 `Promise.all`，这是最基本的性能素养。

---

## 「其实你每天都在用」

**1. 任何带 `await` 的网络请求**

```js
const users = await fetch('/api/users').then(r => r.json());
```

**2. React Server Components / Next.js**

```jsx
export default async function Page() {
  const data = await db.query('SELECT ...');
  return <div>{data}</div>;
}
```

**3. 带重试的请求封装**

```js
async function fetchWithRetry(url, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try { return await fetch(url).then(r => r.json()); }
    catch (e) { if (i === retries - 1) throw e; }
  }
}
```

**4. for-await-of 遍历异步流**

```js
for await (const chunk of stream) {
  process(chunk);
}
```

**5. 动态 import**

```js
const module = await import('./heavy-module.js');
```

---

## 常见误解（FAQ）

**❌ 误区 1：「async 函数里所有代码都是异步的」**

只有 `await` 之后的代码是异步的（微任务）。`await` 之前的代码和 async 函数调用本身都是同步执行的。

**❌ 误区 2：「await 阻塞了主线程」**

`await` 暂停的是**当前 async 函数的执行上下文**，不是主线程。主线程继续跑别的代码。把 await 想象成"我先离开一下，你们继续"。

**❌ 误区 3：「forEach 里写 await 会串行执行」**

既不会串行也不会正确等待。`forEach` 的回调是同步执行的，里面的 `await` 不会阻塞循环——所有请求几乎同时发出，但 `forEach` 本身不等结果。要串行用 `for...of`，要并发用 `Promise.all` + `map`。

**❌ 误区 4：「async 函数比 Promise.then 快」**

本质一样。async/await 编译后就是 Promise 链，性能完全等价。选哪个是代码可读性的问题，不是性能问题。

---

## 一句话总结

**async/await 用"暂停"的假象掩盖了"回调"的真相——但这个假象足够好用，让异步代码读起来像同步，写起来像呼吸。**
