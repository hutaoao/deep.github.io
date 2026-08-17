---
layout: post
title: "Express/Koa原理深度解析"
date: 2026-09-12 00:00:00 +0800
categories: ["服务端", "Node.js"]
tags: [Express, Koa, 中间件, 洋葱模型, compose, 框架原理]
description: >
  两个国民级 Node Web 框架的内核对决：Express 的线性中间件 vs Koa 的洋葱模型，next() 与 await next() 的差异、错误处理与 ctx 设计，面试常问。
---

## 一句话概括

Express 和 Koa 的核心都是**中间件机制**——一个能「处理请求、再决定是否把控制权交给下一个」的函数。区别在于：Express 是**线性**地把中间件一个接一个跑完；Koa 利用 `async/await` 实现了**洋葱模型**，请求从外进、从内出，每个中间件能在「请求前」和「响应前」各插一手。

面试爱问它，因为「中间件」是几乎所有 Node Web 框架的底层范式；搞懂两者的执行模型和错误处理差异，你用 NestJS、Fastify 甚至自己造轮子都不会懵。

## 核心知识点

### 1. 中间件本质：一个「能传递控制权」的函数

无论 Express 还是 Koa，中间件都是 `(ctx/req, res, next) => {}` 这样的函数，核心动作是「干完活，把控制权交给下一个」。区别只在「怎么交」。

### 2. Express：线性执行，next() 把接力棒交下去

Express 把所有中间件/路由排成一个数组，依次执行；`next()` 就是把下标推进到下一个：

```js
class MiniExpress {
  constructor() { this.stack = []; }
  use(fn) { this.stack.push(fn); }
  handle(req, res) {
    let i = 0;
    const next = (err) => {
      if (err) return this._handleError(err, req, res);
      const fn = this.stack[i++];
      if (!fn) return;
      try { fn(req, res, next); } catch (e) { next(e); }
    };
    next();
  }
  _handleError(err, req, res) {
    res.statusCode = 500;
    res.end('err: ' + err.message);
  }
}
```

Express 4 的 `async` 中间件有个大坑——它**不会自动捕获异步错误**，必须手动 `try/catch` 转给 `next(err)`：

```js
// ❌ Express 4 里 async 中间件抛错不会被自动捕获，进程可能崩
app.use(async (req, res, next) => {
  const data = await fetchData(); // 抛错 → 没人 catch
  res.json(data);
});

// ✅ 手动 try/catch 转给 next(err)
app.use(async (req, res, next) => {
  try {
    res.json(await fetchData());
  } catch (e) { next(e); } // Express 5 起可省略，会自动捕获 async 错误
});
```

### 3. Koa：洋葱模型，await next() 让前后都能插手

Koa 的核心是 `compose` 函数：把中间件数组合成一个 Promise 链。`await next()` 相当于「先让内层跑完，再回来执行我后面的代码」：

```js
function compose(middlewares) {
  return (ctx) => {
    let i = -1;
    const dispatch = (n) => {
      if (n <= i) return Promise.reject(new Error('next() 调用多次'));
      i = n;
      const fn = middlewares[n];
      if (!fn) return Promise.resolve();
      return Promise.resolve(fn(ctx, () => dispatch(n + 1)));
    };
    return dispatch(0);
  };
}

// 使用：洋葱模型的执行顺序
const app = compose([
  async (ctx, next) => { console.log(1); await next(); console.log(4); },
  async (ctx, next) => { console.log(2); await next(); console.log(3); },
]);
await app({});
// 输出：1 → 2 → 3 → 4（请求从外到内，响应从内到外）
```

因为返回的是 Promise，中间件里抛错会被 `await` 自然冒泡，由 `app.on('error')` 统一接住，无需手动 `next(err)`。

### 4. 设计哲学差异

| 维度 | Express | Koa |
|---|---|---|
| 中间件模型 | 线性回调 | 洋葱模型（Promise 链） |
| 错误处理 | `next(err)` / 4 参数中间件 | `try/catch` 自动冒泡 |
| 响应对象 | 直接操作 `res.send/json/end` | 统一写 `ctx.body` |
| 路由 | 内置 | 需 `koa-router` 等第三方 |
| async 错误 | 4.x 需手动 catch | 原生支持 |
| 源码体积 | ~2000 行 | ~1000 行 |

Koa 的 `ctx` 用 `Object.create(proto)` 为每个请求创建独立上下文，既隔离又共享原型方法；`ctx.throw(401, 'no token')` 是抛标准化错误的快捷方式。

## 其实你每天都在用

- **`app.use(cors())` / `app.use(express.json())`**：应用级中间件，对所有请求生效
- **`app.get('/user', handler)`**：路由级中间件，只匹配特定方法+路径
- **`app.use((err, req, res, next) => ...)`**：4 参数的错误处理中间件
- **Koa 里 `app.use(async (ctx, next) => { await next(); ctx.body = ... })`**：洋葱的前后置逻辑
- **`koa-bodyparser` / `koa-router`**：社区中间件拼装出完整能力
- **`ctx.throw(401, 'no token')`**：Koa 统一抛出带状态码的错误

## 常见误解（FAQ）

**❌ 误区一：「Express 的 async 中间件抛错会自动被 catch」**

Express 4 不会——async 函数里抛的错不在 `try/catch` 范围内，框架捕获不到，可能直接带崩进程。必须手写 `try/catch` 后调 `next(err)`。Express 5 才内置了 async 错误捕获，写 4.x 代码时别想当然。

**❌ 误区二：「Koa 的 `next()` 和 Express 的 `next()` 一样」**

不一样。Express 的 `next()` 是普通函数、不返回 Promise，调完就继续往下走；Koa 的 `next()` 返回一个 Promise，必须 `await` 它，才能等「内层中间件都跑完」再执行后半段——这正是洋葱模型的关键。忘了 `await`，前置/后置逻辑就串味了。

**❌ 误区三：「洋葱模型就是手写嵌套函数」**

不是。洋葱模型是 `compose` 用**递归 + Promise 链**把扁平的中间件数组串起来的，不是让你手动 `fn1(fn2(fn3()))` 嵌套。理解 `dispatch` 这个递归函数，才算真懂 Koa。

**❌ 误区四：「Koa 比 Express 快很多」**

两者内核量级接近，性能差异主要来自中间件生态和使用方式，而非框架本身。Koa 的优势是「更优雅的异步错误处理 + 更薄的抽象」，不是「快好几倍」。选型看团队习惯和生态，别被「快」带偏。

## 一句话总结

Express 用 `next()` 把中间件**线性接力**跑完，Koa 用 `await next()` 把中间件**卷成洋葱**——前者简单直接、生态庞大，后者前后置对称、异步错误处理优雅；理解了「控制权怎么传」，你就握住了所有 Node Web 框架的命门。
