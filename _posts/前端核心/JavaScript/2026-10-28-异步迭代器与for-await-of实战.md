---
layout: post
title: "异步迭代器与 for-await-of 实战"
date: 2026-10-28 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [异步迭代器, Symbol.asyncIterator, for-await-of, async生成器, 流式处理, Promise]
description: >
  ES2018 异步迭代协议：用 Symbol.asyncIterator 描述"随时间到达的数据流"，for-await-of 消费异步可迭代对象，配合 async 生成器实现分页拉取与流式处理。
---

## 一句话概括

同步迭代协议处理"已经在手里"的数据，可如果数据**一个一个、过一会儿才来**（接口分页、文件流、WebSocket 消息），就需要**异步迭代协议**。核心就三样：
- `Symbol.asyncIterator`（ES2018 引入）：一个对象实现它并返回**异步迭代器**，对象就能被异步遍历；
- **异步迭代器**的 `next()` 返回 `Promise<{ value, done }>`；
- **`for await...of`**：语法上和 `for...of` 一样，但每次 `await` 迭代结果，能消费异步可迭代对象，也能消费同步可迭代对象。

面试为什么问？因为它连接了"迭代协议"和"异步"两个高频考点，而且 `for await...of` 会先找 `Symbol.asyncIterator`、找不到再退化成同步迭代器——这个退化规则是必考易错点。

## 核心知识点

### 1. 异步可迭代协议：给对象加 [Symbol.asyncIterator]

```js
const delays = {
  list: [500, 1300, 3500],
  wait(ms) { return new Promise(r => setTimeout(r, ms)); },
  async *[Symbol.asyncIterator]() {        // ✅ async 生成器最省事
    for (const ms of this.list) {
      await this.wait(ms);
      yield `等了 ${ms}ms ${new Date().toLocaleTimeString()}`;
    }
  },
};

(async () => {
  for await (const msg of delays) console.log(msg);
  // 每 0.5s / 1.3s / 3.5s 各打印一条
})();
```

手动实现（不依赖 async 生成器）也行：`[Symbol.asyncIterator]()` 返回一个带 `next()` 的对象，而 `next()` 返回 `Promise.resolve({ value, done })`。但**实际写几乎都用 `async function*`**，让引擎帮你包 Promise。

### 2. 异步迭代器 vs 同步迭代器：只看 next() 返回什么

| 维度 | 同步迭代器 | 异步迭代器 |
|---|---|---|
| 协议符号 | `Symbol.iterator` | `Symbol.asyncIterator` |
| `next()` 返回 | `{ value, done }` | `Promise<{ value, done }>` |
| 消费语法 | `for...of` | `for await...of` |
| 典型数据 | 内存里已有的数组/集合 | 接口、流、事件这类"随时间到达"的数据 |

一句话记忆：**同步的 `next()` 立刻给值，异步的 `next()` 给一个"将来才给值"的 Promise**。

### 3. for await...of 怎么工作：先 async 后 sync 的退化规则

这是面试必背的细节。`for await...of` 遍历一个对象时：
1. 先找它的 `Symbol.asyncIterator` 方法并调用，拿到异步迭代器；
2. **如果没有 `Symbol.asyncIterator`，就退化去找 `Symbol.iterator`**，把同步迭代器包成异步迭代器（每个 `next()` 结果套一层 Promise）；
3. 循环反复 `await` 迭代器的 `next()`，直到 `done: true`；提前 `break`/抛错会调用迭代器 `return()` 做清理。

```js
// ✅ 普通数组（只有 Symbol.iterator）也能被 for await...of 消费
async function demo() {
  for await (const x of [1, 2, 3]) console.log(x); // 1 2 3
}
```

⚠️ 两个坑：
- `for await...of` **只能在能用 `await` 的地方**（async 函数体内、模块顶层）使用，普通脚本顶层不行；
- 它比 `for...of` 慢一点：即使是同步可迭代对象，每一轮也要 `await` 一次 Promise 解包。

### 4. 实战一：分页接口"一页一页惰性拉"

```js
class PaginatedAPI {
  constructor(url, size = 10) { this.url = url; this.size = size; }
  async *[Symbol.asyncIterator]() {
    let page = 1, hasMore = true;
    while (hasMore) {
      const res = await fetch(`${this.url}?page=${page}&size=${this.size}`);
      const data = await res.json();
      for (const item of data.items) yield item;  // ✅ 每来一项就吐一项，不囤全量
      hasMore = data.hasNext;
      page++;
    }
  }
}
// 用法：逐条处理，内存友好
(async () => {
  for await (const row of new PaginatedAPI('/api/users')) {
    console.log('处理', row.id);
  }
})();
```

### 5. 实战二：消费 ReadableStream（WHATWG 流）

原生 JS 里**没有**默认就是异步可迭代的内置对象（核心语言层面），但 Web 的 `ReadableStream`、Node 的 `Readable` 流实现了 `Symbol.asyncIterator`，所以能直接 `for await...of` 按块读：

```js
async function size(url) {
  const res = await fetch(url);
  let total = 0;
  for await (const chunk of res.body) {  // ✅ res.body 是异步可迭代流
    total += chunk.length;
  }
  return total;
}
```

### 6. 同步生成器里 yield 了 rejected Promise 的陷阱

MDN 明确指出：`for await...of` 消费**同步生成器**时，如果生成器 `yield` 了一个 rejected Promise，循环会在消费它时**抛错、且不会执行生成器内部的 `finally`**——资源可能泄漏。正确姿势是：同步生成器用 `for...of`，异步生成器用 `for await...of`，在循环里自己 `await` 每个 Promise。

```js
// ❌ 危险：同步生成器 yield rejected promise
function* bad() {
  try { yield Promise.reject(new Error('x')); }
  finally { console.log('清理'); } // 这行可能不执行！
}
// ✅ 改为 async 生成器 + for await...of，或循环内显式 await
async function* good() {
  try { yield Promise.reject(new Error('x')); }
  finally { console.log('清理'); } // 正常执行
}
```

## 其实你每天都在用

- **分页拉数据**：用异步生成器把"翻页循环"藏起来，调用方只管 `for await`，不用关心 page 怎么传
- **流式接口 / 大文件**：`fetch(url).then(r => r.body)` 的 body 是异步可迭代流，边下边处理，不占内存
- **WebSocket / SSE 消息**：把"一条条到达的消息"包成异步可迭代对象，用 `for await...of` 消费（配合 `break` 退出）
- **Node.js 读大文件**：`fs.createReadStream` 的流能用 `for await...of` 按 chunk 读
- **async 生成器做状态机 / 后台任务轮询**：每个 `yield` 一次 `await`，天然就是"停一下、等一会儿、再继续"
- **`for await...of` 也能遍历普通数组**：退化规则让它能当"通用遍历"用，但要记得它的 async 上下文限制

## 常见误解（FAQ）

**❌ 误区一："数组、Map 这些默认支持 for await...of"**

半对。它们**默认只实现 `Symbol.iterator`**，`for await...of` 是通过"退化规则"把同步迭代器包成异步来消费的，不是它们原生是异步可迭代。真正异步可迭代的对象要实现 `Symbol.asyncIterator`（如 `ReadableStream`）。

**❌ 误区二："for await...of 和普通 for...of 没区别"**

区别两点：① `for await...of` 能在**同步和异步**可迭代对象上用，普通 `for...of` 只认同步；② 它**只能在 async 函数或模块顶层**用，且每轮都要 `await`，同步数据时用它反而更慢。

**❌ 误区三："我得手写 Promise 版的 next() 才算异步迭代器"**

不用。绝大多数情况用 `async function*`（`async *[Symbol.asyncIterator]()`）就行，引擎自动把每个 `yield` 包成 `Promise<{value,done}>`，手写 Promise 既啰嗦又易错。

**❌ 误区四："异步迭代器和异步生成器是两种东西"**

异步生成器（`async function*`）返回的就是遵守**异步可迭代协议**的对象，是"实现异步迭代器最省事的方式"，不是并列概念。你可以理解成：异步生成器 = 异步迭代器的语法糖。

**❌ 误区五："Symbol.asyncIterator 是 ES2022 才有的"**

记错版本了。它随 **ES2018（ES9）** 一起标准化，主流浏览器自 2020 年 1 月起全面支持。面试说版本别张冠李戴。

## 一句话总结

异步迭代协议 = 一个 `[Symbol.asyncIterator]()` 返回"next() 给 Promise 的迭代器"；`for await...of` 先找 async 再退化 sync、每轮 `await`、只能用在 async/模块里；实战用 `async function*` 写最省事，分页拉取和流处理是它最对味的场景，但同步生成器里 yield rejected Promise 会让 `finally` 不执行，是个真实坑。
