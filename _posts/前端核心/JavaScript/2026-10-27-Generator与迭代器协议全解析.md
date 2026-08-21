---
layout: post
title: "Generator 与迭代器协议全解析"
date: 2026-10-27 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [Generator, 迭代器协议, yield, Symbol.iterator, 协程, 惰性求值]
description: >
  for...of、展开运算符、解构背后的底层协议；Generator 如何用 function* 与 yield 优雅实现可暂停、可双向通信的迭代器，面试高频辨析。
---

## 一句话概括

迭代器（Iterator）协议和生成器（Generator）是 JavaScript "一次取一个值"这套机制的地基。说白了：
- **迭代器**是一个带 `next()` 方法的对象，每次调用返回 `{ value, done }`；
- **可迭代对象**是带 `[Symbol.iterator]()` 方法的对象，它返回一个迭代器；`for...of`、展开运算符 `...`、解构、`yield*` 全都依赖这个协议；
- **生成器（Generator）**是用 `function*` + `yield` 定义的特殊函数，调用后返回一个**既是迭代器又是可迭代对象**的生成器对象，它最大的本事是"能暂停、能恢复、还能双向传值"。

面试为什么爱问？因为这是 `for...of` 的底层，也是手写迭代器的标准答案；更进阶的考点是 Generator 在 async/await 出现之前如何用来写异步流程控制（Redux-Saga、co 库），以及它和 `Promise`/`async` 的区别。

## 核心知识点

### 1. 迭代器协议：一个 next() 走天下

面试官问：迭代器协议到底是什么？
答：就是一个对象实现了 `next()` 方法，每次返回 `{ value, done }`。`done: true` 表示序列结束，之后 `next()` 一般返回 `{ value: undefined, done: true }`。

```js
function createRange(start, end) {
  let current = start;
  return {
    next() {
      if (current <= end) return { value: current++, done: false };
      return { value: undefined, done: true }; // ❌ 容易漏写 done，导致死循环
    },
  };
}
const it = createRange(1, 3);
it.next(); // { value: 1, done: false }
it.next(); // { value: 2, done: false }
it.next(); // { value: 3, done: false }
it.next(); // { value: undefined, done: true }
```

### 2. 可迭代协议：加上 [Symbol.iterator] 才能被 for...of 消费

光有迭代器还不够，要让 `for...of`、展开、`[...x]` 用得上，对象必须实现 `Symbol.iterator` 方法并返回迭代器：

```js
const range = {
  from: 1,
  to: 3,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next() {
        if (current <= last) return { value: current++, done: false };
        return { value: undefined, done: true };
      },
    };
  },
};

// ✅ 现在 range 能被所有"迭代语法"消费
for (const n of range) console.log(n); // 1 2 3
[...range]; // [1, 2, 3]
```

⚠️ 易错点：**普通对象 `{}` 默认不是可迭代的**，不能 `for...of`。内置可迭代对象只有 `String / Array / TypedArray / Map / Set`（以及 `arguments`、NodeList 等类数组视情况），`Object` 不在内。想遍历普通对象要用 `Object.keys/values/entries`，或自己实现协议。

### 3. 生成器：函数也能"暂停"

面试官问：Generator 是什么？和普通函数有什么区别？
答：`function*` 定义生成器函数，调用后**不立即执行**，而是返回一个生成器对象；碰到 `yield` 就暂停并把值交出去，下次调 `next()` 再往下跑。

```js
function* gen() {
  yield 1;
  yield 2;
  return 'done';
}
const g = gen();        // ❌ 注意：调用 gen() 不执行函数体，只拿到生成器对象
g.next(); // { value: 1,    done: false }
g.next(); // { value: 2,    done: false }
g.next(); // { value: 'done', done: true }  // return 会让 done 变 true
g.next(); // { value: undefined, done: true }
```

三个关键点：
- 生成器对象**同时遵守迭代器协议和可迭代协议**（它的 `[Symbol.iterator]()` 返回自身），所以能直接 `for...of`，但**只能迭代一次**；
- 没有箭头的生成器写法（`() => * ()` 不存在），`function` 和 `*` 是分开的两个 token；
- `yield*` 把控制权委托给另一个可迭代对象，相当于把它的每个值都"转交"出来。

### 4. next(value) 双向通信：第一个参数被忽略

这是面试加分项。调用 `next(arg)` 时，`arg` 会成为**当前暂停的 `yield` 表达式的返回值**：

```js
function* talk() {
  const name = yield '你叫什么？';    // 第一次 next() 到这里产出问题
  const color = yield `你好 ${name}`; // 第二次 next('Alice') 把 'Alice' 喂给上一句 yield
  return `${name} 喜欢 ${color}`;
}
const t = talk();
t.next();            // { value: '你叫什么？', done: false }
t.next('Alice');     // { value: '你好 Alice', done: false }  // 'Alice' 成为上一 yield 的值
t.next('蓝色');       // { value: 'Alice 喜欢 蓝色', done: true }
```

⚠️ 记住：**第一次 `next()` 传的参数会被忽略**——因为此时还没停在任何一个 `yield` 上，没有表达式接收它。生成器当协程通信时特别容易踩这个坑。

### 5. return() / throw()：从外面结束或打断生成器

```js
function* g() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log('清理了');
  }
}
const it = g();
it.next();          // { value: 1, done: false }
it.return('结束');   // { value: '结束', done: true }，finally 里的"清理了"会执行
it.throw(new Error('出错')); // 在暂停点抛错，被 try/catch 接住否则向上冒泡
```

一句话：`return(value)` 像在暂停点插了个 `return`，`throw(err)` 像插了个 `throw`。只要生成器内部 `try...finally` 接住，就能做资源清理。

### 6. Generator vs async/await：为什么现在异步都用 async

历史考点：在 `async/await` 普及前，Generator（配合执行器如 co / Redux-Saga）用来写"看起来像同步的异步代码"。但 Generator 本身产出的是同步的 `{ value, done }`，要和 `Promise` 配合才异步。现在 `async/await` 是更简单的官方方案，所以**面试别把 Generator 当异步首选**，它真正的现代价值是"惰性迭代 / 自定义迭代器 / 状态机"。

| 维度 | 生成器 Generator | async/await |
|---|---|---|
| 产出对象 | 同步 `{ value, done }` | Promise |
| 暂停能力 | 有（yield） | 有（await） |
| 异步通信 | 需执行器包装 Promise | 原生支持 |
| 现代主用场景 | 自定义迭代器、惰性序列、状态机 | 异步流程控制 |

## 其实你每天都在用

- **`for...of` 遍历数组/Map/Set**：底层就是反复调 `[Symbol.iterator]().next()`，你每天都在用迭代协议
- **展开运算符 `...` 和数组解构**：`const [a, ...rest] = arr` 也走可迭代协议
- **`yield*` 委托**：写复杂生成器时用它把子生成器的值透传出来
- **Redux-Saga**：用 `function*` 写副作用流程，effect 就是 Generator 的 yield，是"Generator 做异步控制流"的活化石
- **自定义页码 / 区间 / 树遍历**：不想一次性生成大数组，用 Generator 惰性吐值，省内存
- **惰性无限序列**：`function* id() { let i = 0; while (true) yield i++; }` 理论上能表示无限数据

## 常见误解（FAQ）

**❌ 误区一："迭代器和可迭代对象是同一个东西"**

错。迭代器是带 `next()` 的对象；可迭代对象是带 `[Symbol.iterator]()` 方法、返回迭代器的对象。`for...of` 要的是**可迭代对象**。数组是可迭代对象，但数组的 `[Symbol.iterator]()` 返回的才是迭代器——两者角色不同。

**❌ 误区二："调用 gen() 会立刻执行函数体"**

错。调用生成器函数**只创建生成器对象、不跑任何函数体代码**，直到第一次 `next()` 才从函数开头跑到第一个 `yield`。这正是"惰性"的来源。

**❌ 误区三："next() 传的参数能影响第一次 yield"**

错。第一次 `next()` 的参数**永远被忽略**，因为此时还没有挂起的 `yield` 表达式接收它。要从外部初始化，应在生成器函数参数里传，而不是靠第一个 `next()`。

**❌ 误区四："Generator 是用来写异步的，比 async/await 强"**

过时了。Generator 产出的是同步迭代结果，要和 Promise + 执行器才做异步；`async/await` 是原生、更简单的异步方案。现代前端里 Generator 真正的价值是可迭代协议、惰性求值与状态机，不是替代 async。

**❌ 误区五："普通对象能直接 for...of"**

错。`{}` 默认没有 `[Symbol.iterator]`，`for (const k of {})` 直接报错。要遍历对象内容请用 `Object.entries(obj)` 或自己实现可迭代协议。

## 一句话总结

迭代器协议 = 一个 `next()→{value,done}`；可迭代协议 = 一个 `[Symbol.iterator]()`；Generator 用 `function*` + `yield` 把"可暂停、可双向通信的迭代器"一行行写出来——它既是迭代器也是可迭代对象，但记住它产出的是同步结果、第一次 `next()` 参数会被忽略、普通对象默认不可迭代。
