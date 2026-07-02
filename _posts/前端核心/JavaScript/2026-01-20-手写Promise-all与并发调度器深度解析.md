---
title: "手写Promise.all与并发调度器深度解析"
date: 2026-01-20
categories: [前端核心, JavaScript]
tags: [Promise.all, Promise.race, 并发调度器, 手写实现, JavaScript]
---

## 一句话概括

手写 `Promise.all` / `Promise.race` / `Promise.allSettled` 的底层只用了一个计数器 + 一个 settled 标志，而并发调度器不过是在这层机制上套了一个 `while (running < max && queue.length)` 的消费循环——理解这四个字就理解了 Promise 组合的全部。

## 核心知识点

### ① Promise.all：计数器决定何时 resolve

全成功才成功，一失败即拒绝。核心只有两点：**结果按索引写入**（不是按完成顺序） + **计数器满即 resolve**。

```javascript
function myAll(iterable) {
  return new Promise((resolve, reject) => {
    const arr = [...iterable];
    if (arr.length === 0) return resolve([]);

    const results = new Array(arr.length);
    let count = 0; // ← 这就是核心

    arr.forEach((item, i) => {
      Promise.resolve(item).then(
        val => {
          results[i] = val; // 写入对应索引，不是 push
          if (++count === arr.length) resolve(results);
        },
        reason => reject(reason) // 谁先挂，整个 all 就挂
      );
    });
  });
}

// 验证
myAll([Promise.resolve(1), 2, Promise.resolve(3)]).then(console.log); // [1,2,3]
myAll([Promise.reject('boom'), Promise.resolve(1)]).catch(console.log); // 'boom'
```

### ② Promise.race：谁先 settled 谁赢

race 没有计数器，只有一个 `settled` 锁——第一个落地的结果一把锁锁死，后面的全部忽略。

```javascript
function myRace(iterable) {
  return new Promise((resolve, reject) => {
    const arr = [...iterable];
    if (arr.length === 0) return; // 空 → 永远 pending

    let settled = false;

    arr.forEach(item => {
      Promise.resolve(item).then(
        val => { if (!settled) { settled = true; resolve(val); } },
        err => { if (!settled) { settled = true; reject(err); } }
      );
    });
  });
}

// 验证
myRace([
  new Promise(r => setTimeout(() => r('slow'), 500)),
  new Promise(r => setTimeout(() => r('fast'), 50)),
]).then(console.log); // 'fast'
```

### ③ Promise.allSettled：永远不 reject 的 all

跟 `myAll` 几乎一样，唯一区别：`.then()` 的两个分支都记录结果并 `++count`，`resolve` 和 `reject` 殊途同归。

```javascript
function myAllSettled(iterable) {
  return new Promise(resolve => {
    const arr = [...iterable];
    if (arr.length === 0) return resolve([]);

    const results = new Array(arr.length);
    let count = 0;

    arr.forEach((item, i) => {
      Promise.resolve(item).then(
        value => { results[i] = { status: 'fulfilled', value }; if (++count === arr.length) resolve(results); },
        reason => { results[i] = { status: 'rejected', reason }; if (++count === arr.length) resolve(results); }
      );
    });
  });
}

// 验证
myAllSettled([Promise.resolve(1), Promise.reject('err'), Promise.resolve(3)])
  .then(r => console.log(r.map(x => x.status))); // ['fulfilled', 'rejected', 'fulfilled']
```

### ④ 并发调度器：while 循环 + 任务队列

调度器本质是一个**消费者循环**：只要并发没满且队列非空，就出队执行。任务完成后 `finally` 里递归调用自己继续消费。

```javascript
class ConcurrencyScheduler {
  constructor(limit = 3) {
    this.limit = limit;
    this.queue = [];
    this.running = 0;
  }

  add(task) {
    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject });
      this.#next();
    });
  }

  #next() {
    while (this.running < this.limit && this.queue.length) {
      const { task, resolve, reject } = this.queue.shift();
      this.running++;
      Promise.resolve(task())
        .then(resolve, reject)
        .finally(() => { this.running--; this.#next(); });
    }
  }
}

// 验证：5个任务并发3个 → 1/2/3 同时启动，300ms后 4/5 接着启动
const s = new ConcurrencyScheduler(3);
for (let i = 1; i <= 5; i++) s.add(() => new Promise(r => setTimeout(() => r(i), 300)));
```

### ⑤ Promise.any：与 all 镜像对称（ES2021）

```javascript
function myAny(iterable) {
  return new Promise((resolve, reject) => {
    const arr = [...iterable];
    if (arr.length === 0) return reject(new AggregateError([], 'All rejected'));
    const errors = new Array(arr.length);
    let count = 0;
    arr.forEach((item, i) =>
      Promise.resolve(item).then(resolve, err => {
        errors[i] = err;
        if (++count === arr.length) reject(new AggregateError(errors, 'All rejected'));
      })
    );
  });
}
```

## 其实你每天都在用

**① 首屏数据聚合。** 同时请求用户信息、购物车、消息未读、首页推荐，等全部返回再渲染 → `Promise.all`。只关心最快显示一块内容 → `Promise.race`。

**② 图片批量上传。** 朋友圈选 9 张图，后端限制同时 3 个连接。不能无脑发 9 个请求——并发调度器上场：3 个一批，3 个一批，有序消耗连接。

**③ 请求降级与超时。** 同时请求主备两个支付接口，哪个先返回用哪个 → `Promise.race`。必须等所有子服务返回再汇总成败 → `Promise.allSettled`。

**④ 文件分片下载。** 大文件切 10 个 chunk，浏览器同域名 TCP 连接上限 6，用调度器控制在 4 并发——超了 TCP 窗口也扛不住。

**⑤ 表单多字段校验。** 用户名/手机号/邮箱三个异步校验，任何一个失败就阻止提交 → `Promise.all`。

## 常见误解

**❌ 误区：`Promise.all` 里一个 reject，其他 Promise 会自动取消。**

不会。Promise 一旦创建就无法取消，其他任务会照常执行到底，只是外层 all 已经 reject 了，它们的结果被丢弃。真正需要取消能力的，要用 `AbortController` 或任务内部检查取消标志。

**❌ 误区：`Promise.all([p1, p2, p3])` 结果数组的顺序是完成顺序。**

不是。结果按**输入顺序**排列，跟谁先跑完无关。你可以在 `p1.then` 里记 `results[0]`、`p2.then` 里记 `results[1]`——索引由传入时的位置决定，不是按 `.then()` 触发的先后。

**❌ 误区：`Promise.race([])` 会 reject 或返回 null。**

不会。空数组的 race 会**永远 pending**——因为没有任何 Promise 来触发 resolve 或 reject。这是一个经典的面试陷阱：`Promise.all([])` → 立即 resolved `[]`，`Promise.race([])` → 永远悬停。

**❌ 误区：调度器里用 `setInterval` 轮询队列最方便。**

轮询是浪费。正确的做法是**事件驱动**：每个任务在 `.finally()` 里调用 `#next()` 自动触发下一个，没有任务跑的时候什么都不做，CPU 零消耗。`setInterval` 即使在空闲时也在消耗 CPU 时钟周期。

## 一句话总结

Promise 组合方法的本质不是一种 API，而是一种**状态机**——计数器追踪进度，settled 锁防止竞态，`while` 循环消费队列；手写完这三个模式，你对 JavaScript 异步的理解就跨越了"会用 API"到"理解事件循环"的分水岭。
