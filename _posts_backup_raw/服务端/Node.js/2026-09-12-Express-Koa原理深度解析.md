---
layout: post
title: "Express/Koa原理深度解析"
date: 2026-09-12 00:00:00 +0800
categories: ["服务端", "Node.js"]
tags: [Express, Koa, 中间件, 洋葱模型]
math: true
mermaid: true
---

## 一句话概括

Express 和 Koa 是 Node.js 最流行的两个 Web 框架，它们的核心都是中间件机制——Express 采用线性中间件执行模型，Koa 则使用洋葱模型，本文将从源码层面剖析两者的中间件调度原理、路由实现、错误处理机制以及设计哲学差异。

## 背景与意义

在 Node.js 诞生之初，用原生的 `http.createServer` 直接处理请求是件非常原始的事情：
- 路由需要手动解析 `req.url` 来判断
- 每个中间功能（日志、解析 body、设置 CORS 头）需要层层嵌套回调
- 错误处理分散在各处

这正是 Express 出现的原因——它提供了一套简单而强大的中间件系统，让 Web 应用开发变得有章可循。随后 Koa 在 Express 的基础上更进一步，利用 ES6 的 `async/await` 实现了更优雅的洋葱模型中间件。

可以说，Express 定义了 Node.js Web 开发的「中间件模式」，而 Koa 则是这种模式在 async/await 时代的完美进化。理解两者的差异和原理，能让你在使用任意 Node.js Web 框架时都游刃有余。

## 概念与定义

**中间件（Middleware）**：一个函数，接收请求对象、响应对象和下一个中间件函数作为参数，可以对请求/响应对象进行修改或在处理完成后调用下一个中间件。

**洋葱模型（Onion Model）**：Koa 中间件的执行模型，中间件像洋葱的层次一样层层嵌套，请求从外层进入、从内层穿出，每个中间件可以在请求前后各执行一次代码。

**`next()` 函数**：在中间件中调用以将控制权传递给下一个中间件的函数。

**应用级中间件**：通过 `app.use()` 注册的中间件，对所有请求生效。

**路由级中间件**：绑定到特定路径的中间件，如 `app.get('/user', handler)`。

## 核心知识点拆解

### 1. Express 中间件机制：线性执行

Express 的核心源码只有约 2000 行，其中中间件管理的逻辑占据半壁江山。中间件本质上是一个函数数组，每个请求到来时依次执行。

```javascript
/**
 * Express 中间件系统的简化实现
 * 展示中间件队列的管理和调度
 */
class MiniExpress {
  constructor() {
    this.middlewares = [];
    this.routes = [];
  }

  // 注册中间件
  use(path, handler) {
    if (typeof path === 'function') {
      // app.use(fn) 形式
      handler = path;
      path = '/';
    }
    
    this.middlewares.push({
      path,
      handler,
      type: 'middleware'
    });
  }

  // 注册路由
  get(path, ...handlers) {
    this.routes.push({
      method: 'GET',
      path,
      handlers
    });
  }

  post(path, ...handlers) {
    this.routes.push({
      method: 'POST',
      path,
      handlers
    });
  }

  // 处理传入的 HTTP 请求
  handleRequest(req, res) {
    const stack = [];
    
    // 将适用的中间件加入执行栈
    for (const mw of this.middlewares) {
      if (req.url.startsWith(mw.path)) {
        stack.push(mw.handler);
      }
    }
    
    // 将适用的路由处理器加入执行栈
    for (const route of this.routes) {
      if (route.method === req.method && route.path === req.url) {
        stack.push(...route.handlers);
      }
    }
    
    // 错误处理中间件（4个参数）
    let errorHandler = null;
    
    // 执行中间件栈
    let index = 0;
    const next = (err) => {
      if (err) {
        // 错误发生时，跳过普通中间件，寻找错误处理中间件
        return handleError(err);
      }
      
      const middleware = stack[index++];
      if (!middleware) return;
      
      try {
        middleware(req, res, next);
      } catch (error) {
        handleError(error);
      }
    };
    
    const handleError = (err) => {
      // 查找错误处理中间件（4 个参数的函数）
      for (const mw of stack) {
        if (mw.length === 4) {
          return mw(err, req, res, next);
        }
      }
      // 没有错误处理中间件，返回 500
      res.statusCode = 500;
      res.end('Internal Server Error');
    };
    
    next();
  }
}

// 使用示例
const app = new MiniExpress();

// 中间件
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

app.use((req, res, next) => {
  const start = Date.now();
  next();
  const duration = Date.now() - start;
  console.log(`请求耗时: ${duration}ms`);
});

app.get('/hello', (req, res, next) => {
  res.end('Hello World');
});

// 错误处理中间件
app.use((err, req, res, next) => {
  console.error('错误:', err.message);
  res.statusCode = 500;
  res.end('Something broke!');
});
```

**Express 中间件执行特点**：
- 执行是**线性的**——从第一个中间件到最后一个中间件依次执行
- 每个中间件执行完后，在 `next()` 之前和之后都可以执行代码
- 但 Express 的 `next()` 之后的代码执行时机存在问题——如果 `next()` 后的代码在下一个中间件处理完之前执行，就会导致类似「响应已经发出，计时才刚开始」的问题

### 2. Koa 洋葱模型：由 async/await 驱动的穿透式执行

Koa 的中间件系统利用 `async/await` 和 Promise 实现了真正的洋葱模型——每个中间件可以「穿透」到下一层，并在返回时执行「后半段」代码。

```javascript
/**
 * Koa 洋葱模型中间件系统的简化实现
 * 核心：compose 函数——中间件的合成引擎
 */
class MiniKoa {
  constructor() {
    this.middlewares = [];
  }

  use(fn) {
    this.middlewares.push(fn);
    return this;
  }

  /**
   * 核心 compose 函数
   * 将中间件数组合并为一个执行链
   * 这是 Koa 中间件机制的精髓
   */
  compose(middlewares) {
    return (context, next) => {
      let index = -1;
      
      return dispatch(0);
      
      function dispatch(i) {
        if (i <= index) {
          return Promise.reject(new Error('next() called multiple times'));
        }
        
        index = i;
        let fn = middlewares[i];
        
        if (i === middlewares.length) {
          fn = next; // 没有更多中间件了，执行可选的最终 next
        }
        
        if (!fn) {
          return Promise.resolve();
        }
        
        try {
          // 关键：返回 Promise 并传入 dispatch(i+1) 作为 next
          return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
        } catch (err) {
          return Promise.reject(err);
        }
      }
    };
  }

  // 模拟处理请求
  async handleRequest(req, res) {
    const ctx = {
      req,
      res,
      request: req,
      response: res,
      body: null,
      status: 200,
      method: req.method,
      url: req.url,
    };
    
    const fn = this.compose(this.middlewares);
    
    try {
      await fn(ctx);
      // 响应
      if (ctx.body) {
        res.end(ctx.body);
      }
    } catch (err) {
      res.statusCode = err.statusCode || 500;
      res.end(err.message || 'Internal Server Error');
    }
  }
}

// 使用示例
const app = new MiniKoa();

// 日志中间件
app.use(async (ctx, next) => {
  console.log(`[请求开始] ${ctx.method} ${ctx.url}`);
  const start = Date.now();
  
  await next(); // 进入下一个中间件
  
  const ms = Date.now() - start;
  console.log(`[请求结束] ${ctx.method} ${ctx.url} - ${ms}ms`);
});

// 计时中间件
app.use(async (ctx, next) => {
  const start = Date.now();
  
  await next();
  
  const ms = Date.now() - start;
  ctx.set && ctx.set('X-Response-Time', `${ms}ms`);
});

// 响应中间件
app.use(async (ctx, next) => {
  ctx.body = 'Hello from MiniKoa';
});

// 执行流程：
// [请求开始] GET /
//    ↓
// 进入计时中间件
//    ↓
// 进入响应中间件 → 设置 ctx.body
//    ↓
// 回到计时中间件（await next() 之后）
//    ↓
// [请求结束] GET / - 2ms
```

**洋葱模型的核心优势**：每个中间件都可以「在请求处理前做点什么」和「在响应发送前做点什么」，比如：
- 请求前：解析 Cookie、验证令牌、记录日志
- 请求后：计算耗时、格式化响应、清理资源

### 3. Express 和 Koa 中间件的本质差异

```javascript
/**
 * 对比 Express 和 Koa 的中间件模型
 */

// Express 模型：手动传递错误
// Express 4.x 中，next() 不会返回 Promise
// 所以 async 中间件的错误必须自己 catch
app.use(async (req, res, next) => {
  try {
    await someAsyncOperation();
    next();
  } catch (err) {
    next(err); // 必须手动传递错误
  }
});

// Koa 模型：Promise 链自动传播错误
// 因为 compose 返回的是 Promise，await 会自然抛出
app.use(async (ctx, next) => {
  await someAsyncOperation(); // 如果抛出异常，自动被捕获
  await next();
});

// 全局错误处理
app.on('error', (err) => {
  console.error('全局错误:', err);
});

/**
 * 演示两者在异步操作处理上的区别
 */
function expressVsKoaAsync() {
  // Express 中的异步陷阱
  const expressApp = require('express')();
  
  // ❌ 这样写是不安全的！
  expressApp.use(async (req, res, next) => {
    // 异步中间件
    const data = await fetchData(); // 如果这里抛错，next 不会自动调用
    req.data = data;
    next();
  });
  // 上面的错误永远不会被 Express 捕获到！
  
  // ✅ 正确的 Express 写法
  expressApp.use((req, res, next) => {
    fetchData()
      .then(data => {
        req.data = data;
        next();
      })
      .catch(next);
  });
  
  // 或者用 async/await + try/catch
  expressApp.use(async (req, res, next) => {
    try {
      req.data = await fetchData();
      next();
    } catch (err) {
      next(err);
    }
  });
}
```

## 实战案例

### 实现一个兼容 Express 和 Koa 模式的中间件调度器

```javascript
/**
 * 完整的中间件调度器实现
 * 同时支持 Express 风格和 Koa 风格的中间件
 */
class MiddlewareEngine {
  constructor() {
    this._stack = [];
    this._errorHandlers = [];
  }

  /**
   * 注册中间件
   * @param {Function|string} path 路径或处理函数
   * @param {Function} handler 处理函数
   */
  use(path, handler) {
    if (typeof path === 'function') {
      handler = path;
      path = '/';
    }
    
    this._stack.push({
      path,
      handler,
      isErrorHandler: handler.length === 4, // 4 个参数 = 错误处理中间件
    });
    
    return this;
  }

  /**
   * Express 风格的线性执行
   */
  runExpressStyle(req, res) {
    let index = -1;
    
    const next = (err) => {
      index++;
      
      // 跳过路径不匹配的中间件
      while (index < this._stack.length) {
        const layer = this._stack[index];
        
        if (err && !layer.isErrorHandler) {
          // 有错误时跳过普通中间件
          index++;
          continue;
        }
        
        if (!err && layer.isErrorHandler) {
          // 无错误时跳过错误处理中间件
          index++;
          continue;
        }
        
        // 检查路径匹配
        if (req.url.startsWith(layer.path)) {
          try {
            if (err) {
              // 错误处理中间件
              layer.handler(err, req, res, () => next(err));
            } else {
              layer.handler(req, res, next);
            }
            return;
          } catch (e) {
            next(e);
            return;
          }
        }
        
        index++;
      }
    };
    
    next();
  }

  /**
   * Koa 风格的洋葱模型执行
   */
  async runKoaStyle(context) {
    const stack = this._stack.filter(l => !l.isErrorHandler);
    
    const dispatch = (i) => {
      if (i >= stack.length) return Promise.resolve();
      
      const layer = stack[i];
      
      try {
        const result = layer.handler(context, () => dispatch(i + 1));
        return Promise.resolve(result);
      } catch (err) {
        return Promise.reject(err);
      }
    };
    
    try {
      await dispatch(0);
    } catch (err) {
      // 尝试错误处理中间件
      for (const handler of this._errorHandlers) {
        await handler(err, context);
      }
    }
  }
}

// 使用示例：中间件调度器
const engine = new MiddlewareEngine();

// 日志中间件
engine.use('/api', (req, res, next) => {
  console.log('Express 模式:', req.url);
  next();
});

// 认证中间件
engine.use('/api', async (ctx, next) => {
  console.log('Koa 模式:', ctx.url);
  await next();
});

// 错误处理
engine.use((err, req, res, next) => {
  console.error('Error:', err.message);
  res.statusCode = 500;
  res.end('Error');
});
```

## 底层原理

### Express 的路由系统（基于 path-to-regexp）

Express 的路由系统基于 `path-to-regexp` 库，将路径字符串转换为正则表达式。

```javascript
/**
 * Express 路由匹配的简化实现
 */
const pathToRegexp = require('path-to-regexp'); // Express 内部依赖

class RouterLayer {
  constructor(method, path, handlers) {
    this.method = method;
    this.handlers = Array.isArray(handlers) ? handlers : [handlers];
    
    // 将路径表达式编译为正则
    this.keys = [];
    this.regexp = pathToRegexp(path, this.keys);
  }

  match(method, path) {
    if (this.method && this.method !== method) {
      return null;
    }
    
    const match = this.regexp.exec(path);
    if (!match) return null;
    
    // 提取路由参数
    const params = {};
    this.keys.forEach((key, i) => {
      params[key.name] = match[i + 1];
    });
    
    return params;
  }
}

// 使用
const layer = new RouterLayer('GET', '/users/:id', [handler]);
const params = layer.match('GET', '/users/42');
console.log(params); // { id: '42' }
```

### Koa 的 compose 与 Promise 链

Koa 的核心在于 `koa-compose` 包，它负责将中间件数组「串」成一个 Promise 链。洋葱模型的本质是：

```
中间件 1 开始
  ├── await next() → 进入中间件 2
  │     ├── await next() → 进入中间件 3
  │     │     ├── await next() → 没有更多中间件，resolve
  │     │     └── 中间件 3 的「后半段」
  │     └── 中间件 2 的「后半段」
  └── 中间件 1 的「后半段」
```

每个 `await next()` 实际上是在等待内层中间件的 Promise resolve，这就是为什么 Koa 能实现「请求进来时从外层到内层，响应出去时从内层到外层」的穿透效果。

### 两者的设计哲学差异

| 维度 | Express | Koa |
|------|---------|-----|
| 中间件模型 | 线性回调 | 洋葱模型（Promise） |
| 错误处理 | 通过 next(err) 或 app.use(4参数) | try/catch 自动传播 |
| 响应对象 | 直接操作 res（send/end/json） | 通过 ctx.body 统一管理 |
| 路由 | 内置 | 需通过 koa-router（第三方） |
| 社区生态 | 极其丰富 | 较丰富 |
| 学习曲线 | 低（直接操作 req/res） | 中（需要理解 ctx 和洋葱模型） |
| async/await | 需手动处理 | 原生支持 |
| 源码体积 | ~2000 行 | ~1000 行 |

## 高频面试题解析

### 面试题1：Express 的 next() 之后还能执行代码吗？

**答**：可以，但需要小心使用。Express 的中间件在 `next()` 之后确实可以写代码，但因为 Express 不强制 `await next()`（也不返回 Promise），`next()` 之后的代码会在**下一个中间件同步执行完后**立即执行，而不是等整个请求处理链完成后再执行。

```javascript
app.use((req, res, next) => {
  console.log('1'); // 先输出
  next();
  console.log('3'); // 在下一个中间件同步执行后立即输出
});

app.use((req, res, next) => {
  console.log('2'); // 同步执行
  res.send('Hello');
});

// 输出：1, 2, 3
```

如果下一个中间件有异步操作，Express 不会等待它完成，所以 `next()` 之后的代码可能在一个「请求已处理」的上下文中执行。

### 面试题2：Koa 的 ctx 是如何创建的？

**答**：Koa 为每个请求创建了一个新的上下文对象 `ctx`，它是 Node.js 的 `req` 和 `res` 对象的封装，并提供了许多便捷属性和方法。最重要的是，Koa 通过 `Object.create(context)` 创建上下文，既保证了每个请求有独立的 ctx，又共享了原型链上的方法。

```javascript
// Koa 源码简化
class Application {
  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    
    return context;
  }
}
```

### 面试题3：Express 如何处理 async 中间件的错误？

**答**：Express 4.x 不支持自动捕获 async 中间件的错误。如果 async 函数中抛出了异常，必须手动 `try/catch` 后调用 `next(err)`。Express 5.x（测试版）已经解决了这个问题，会自动捕获 async 中的异常。

```javascript
// Express 4 - 必须手动处理
app.use(async (req, res, next) => {
  try {
    const data = await fetchData();
    req.data = data;
    next();
  } catch (err) {
    next(err);
  }
});

// Express 5 - 自动捕获
app.use(async (req, res, next) => {
  const data = await fetchData(); // 自动 catch
  req.data = data;
  next();
});
```

### 面试题4：Express 的 Router 和 app 有什么区别？

`express.Router()` 创建一个独立的「迷你应用」，它有自己的中间件栈和路由。`app.use('/api', router)` 可以将多个 router 挂载到不同的路径上，实现了路由模块化。从源码角度看，Router 实际上是 Application 的子集——只包含路由和中间件管理功能，没有监听端口等能力。

### 面试题5：Koa 的 ctx.throw 和 ctx.assert 如何工作？

```javascript
// ctx.throw 是一个快捷的错误抛出方法
ctx.throw(400, 'Bad request');
// 等价于：
const err = new Error('Bad request');
err.status = 400;
err.expose = true;
throw err;

// ctx.assert 类似于 Node.js 的 assert
ctx.assert(ctx.query.name, 400, 'Name is required');
// 如果 ctx.query.name 为 falsy，相当于 ctx.throw(400, 'Name is required')
```

Koa 的全局错误处理通过 `app.on('error', handler)` 监听，这些错误不会导致应用崩溃，而是被中间件系统捕获后触发 `error` 事件。

## 总结与扩展

Express 和 Koa 代表了 Node.js Web 框架发展的两个时代：Express 定义了「中间件」这个模式，Koa 则将其推向了 Promise/async 时代。两者没有绝对的好坏之分：

- **Express**：大而全、社区生态丰富、适合快速开发和大多数传统项目
- **Koa**：小巧优雅、底层控制力强、适合需要精细控制的复杂应用

**扩展方向**：
- **NestJS**：基于 Express（或 Fastify）的现代框架，引入了 Angular 风格依赖注入和装饰器，适合大型企业级应用
- **Fastify**：性能导向的框架，号称比 Express 快 2-3 倍，支持 JSON Schema 验证和自动文档生成
- **中间件模式的安全问题**：静态文件服务、CSRF、XSS 防护等中间件的实现原理
- **Koa 的下一代**：与 Deno 的 `oak` 框架对比，了解中间件模式的跨平台演化
