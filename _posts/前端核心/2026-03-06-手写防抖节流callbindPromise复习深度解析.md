---
layout: post
title: "手写防抖节流callbindPromise复习"
date: 2026-03-06
tags: [JavaScript, 防抖, 节流, call, bind, Promise]
categories: [前端核心, JavaScript]
---

## 一句话概括

防抖与节流是高频触发事件的两种优化策略，call/apply/bind 是 JavaScript 中显式绑定 this 的核心 API，而 Promise 则是异步编程的基石——手写这些函数不仅是面试中反复出现的高频考点，更是深入理解 JavaScript 执行机制、作用域链、闭包、this 指向及异步流程控制的必经之路。

## 背景与意义（为何这些手写题是面试常考内容？）

在前端面试中，「手写XXX」几乎是每一轮技术面绕不开的环节。究其原因，并非面试官想考查候选人的记忆力，而是因为这些「手写题」背后所牵涉的知识点构成了大前端工程师的核心能力模型。

**防抖与节流**是性能优化的基本功。一个真实场景下的搜索输入框，用户每按键一次就发一次请求，会在极短时间内产生数十次无效的 API 调用——这种「抖动」造成的资源浪费在大型应用中可能积累成巨大的服务器成本。面试者能否清晰区分防抖和节流，能否写出带 `immediate` 参数的防抖、能写出时间戳版和定时器版两种节流，直接反映其对「事件循环」「闭包」「时间精度」的理解层次。

**call/apply/bind** 触及的是 JavaScript 最独特也最令人困惑的特性：`this` 绑定。这三大函数是显式控制 `this` 的核心手段。能手写 `call` 意味着你理解函数调用的本质是通过 `this` 来链路执行；能写 `bind` 的 `new` 兼容版本意味着你理解了原型链和构造函数的 new 绑定规则优先级。

**Promise** 则是现代 JavaScript 异步编程的基石。从回调地狱到 `async/await`，中间最关键的一跃就是 Promise。能完整实现一个符合 Promises/A+ 规范的 Promise（包括状态机、`then` 链式调用、值的穿透、异步调度），说明你对异步执行的微任务队列、回调注册与触发时机有深刻理解。

这些手写题的价值不在于「背代码」，而在于通过复刻这些 API 的底层机制，真正理解它们的设计哲学和实现约束。当你写过一遍自己的 Promise 之后，再去看 `Promise.all`、`Promise.race`、`async/await` 的语法糖，会有一种「茅塞顿开」的感觉。

## 概念与定义

### 防抖（Debounce）

**定义**：在事件被触发 n 秒后再执行回调，如果在这 n 秒内又被触发，则重新计时。

**核心思想**：合并一段时间内的多次触发为一次执行。最后一次（或第一次）才是有效的。

- **非 immediate（滞后执行）**：连续触发时，只执行最后一次触发的回调。适合搜索输入框等场景，用户停止输入后才发送请求。
- **immediate（立即执行）**：连续触发时，立即执行第一次触发的回调，但在冷却期内再次触发不会生效。适合按钮防连点等场景，第一次点击立即生效，后续点击被忽略直到冷却结束。

### 节流（Throttle）

**定义**：规定在一个单位时间内，只能触发一次函数。如果这个单位时间内触发多次函数，只有一次生效。

**核心思想**：稀释执行频率，保证一段时间内至少执行一次回调。

- **时间戳版**：通过比较当前时间戳与上次执行时间戳的差值来判断是否执行。第一次触发立即执行。
- **定时器版**：通过定时器来控制执行周期。第一次触发延迟执行（等到定时器到期才执行）。

### call/apply/bind

**call(thisArg, arg1, arg2, ...)**：调用一个函数，其 `this` 指向 `thisArg`，参数列表传入，函数**立即执行**。

**apply(thisArg, [argsArray])**：与 `call` 类似，但参数以数组形式传入，函数**立即执行**。

**bind(thisArg, arg1, arg2, ...)**：返回一个新函数，其 `this` 永久绑定到 `thisArg`，函数**不立即执行**。新函数可作为构造函数（`new` 调用），此时 `this` 绑定失效，按照 `new` 绑定规则处理。

### Promise

**定义**：Promise 是一个对象，代表一个异步操作的最终完成或失败及其结果值。

**三种状态**：
- `pending`（等待中）：初始状态，既没有被兑现，也没有被拒绝。
- `fulfilled`（已兑现）：操作成功完成。
- `rejected`（已拒绝）：操作失败。

**核心机制**：
- 状态一旦变更就不可再变（pending → fulfilled 或 pending → rejected）
- `then` 方法注册回调，支持链式调用
- `then` 的返回值如果是 Promise，则等待其 resolve；如果是普通值，则作为下一个 then 的参数
- 值的穿透：`.then().then().then(v => console.log(v))` 能拿到最终值

## 最小示例 - 每个函数的简单使用

```javascript
// === 防抖 ===
function handleInput(e) {
  console.log('发送搜索请求:', e.target.value);
}
const debouncedInput = debounce(handleInput, 500);
// 用户连续输入时，只有最后一次输入后等待500ms无新输入才触发

// === 节流 ===
function handleScroll() {
  console.log('计算滚动位置');
}
const throttledScroll = throttle(handleScroll, 200);
// 滚动过程中每200ms至多触发一次

// === call ===
function greet(prefix, suffix) {
  console.log(prefix + this.name + suffix);
}
greet.call({ name: 'Alice' }, 'Hello, ', '!'); // Hello, Alice!

// === apply ===
greet.apply({ name: 'Bob' }, ['Hi, ', '?']); // Hi, Bob?

// === bind ===
const boundGreet = greet.bind({ name: 'Charlie' }, 'Hey, ');
boundGreet('~'); // Hey, Charlie~

// === Promise ===
const p = new Promise((resolve) => {
  setTimeout(() => resolve('成功!'), 1000);
});
p.then((val) => console.log(val)); // 1秒后输出: 成功!

// === Promise.all ===
Promise.all([Promise.resolve(1), Promise.resolve(2)])
  .then(([a, b]) => console.log(a + b)); // 3

// === Promise.race ===
Promise.race([
  new Promise(r => setTimeout(() => r('快'), 100)),
  new Promise(r => setTimeout(() => r('慢'), 500))
]).then(console.log); // 快
```

## 核心知识点拆解

### 1. 防抖与节流

#### 1.1 防抖（非 immediate 版）

```javascript
/**
 * 防抖函数（非立即执行版）
 * @param {Function} fn 需要防抖的函数
 * @param {number} delay 延迟时间（毫秒）
 * @param {boolean} [immediate=false] 是否立即执行
 * @returns {Function} 防抖处理后的函数
 */
function debounce(fn, delay, immediate = false) {
  // 利用闭包保存定时器ID
  let timer = null;

  return function (...args) {
    // 保存当前的 this 上下文
    const context = this;

    // 如果已有定时器，清除并重新计时
    if (timer) clearTimeout(timer);

    if (immediate) {
      // 立即执行版本
      // 如果 timer 为 null，说明冷却期已过，立即执行
      const callNow = !timer;
      timer = setTimeout(() => {
        timer = null; // 冷却期结束
      }, delay);
      if (callNow) {
        fn.apply(context, args);
      }
    } else {
      // 非立即执行版本：延迟执行
      timer = setTimeout(() => {
        fn.apply(context, args);
        timer = null;
      }, delay);
    }
  };
}
```

**执行流程分析**：

非 immediate 版的核心是「推迟到冷静期结束后再执行」。当用户连续点击时，每次点击都清空了上一次的定时器并重新开始计时，这样前 n-1 次事件的回调永远不会被执行，只有最后一次事件发生且度过 delay 毫秒后，回调才真正触发。这种模式非常适合「用户停止操作后我才干活」的场景。

`timer` 变量保存在闭包中，每次调用防抖函数时都能访问到同一个 `timer`。`clearTimeout` 并不会把 `timer` 设为 `null`，它只是取消了这个定时器的执行，所以需要判断 `timer` 是否为 `null` 来决定是否立即执行（immediate 模式的关键逻辑）。

#### 1.2 防抖（immediate 版）

immediate 版的区别在于：**先执行，再计时期**。

第一次触发时 `timer` 为 `null`，满足 `callNow = !timer`（即为 `true`），立即执行 `fn`，然后设置一个定时器。在 delay 毫秒内，`timer` 不为 `null`，所以后续触发只会重置定时器，不会执行 `fn`。直到 delay 毫秒后定时器执行，将 `timer` 重置为 `null`，下一次触发才会再次立即执行。

**典型场景**：提交按钮防连点。用户点击按钮后立即发起请求，同时开启冷却期，冷却期内无论用户怎么点都不会重复提交。

#### 1.3 节流（时间戳版）

```javascript
/**
 * 节流函数（时间戳版）
 * 特点：第一次触发立即执行
 * @param {Function} fn 回调函数
 * @param {number} delay 间隔时间（毫秒）
 * @returns {Function} 节流处理后的函数
 */
function throttleTimestamp(fn, delay) {
  // 上次执行时间，初始为 0 确保第一次立即执行
  let previous = 0;

  return function (...args) {
    const context = this;
    const now = Date.now();

    // 如果距离上次执行超过了 delay，则执行
    if (now - previous > delay) {
      fn.apply(context, args);
      previous = now; // 更新上次执行时间
    }
  };
}
```

**执行流程分析**：每次事件触发时，计算当前时间与上次执行时间的差值。如果超过 `delay` 就执行，否则不做任何操作。第一次触发时 `previous` 为 0，`now - 0 > delay` 恒成立，所以立即执行。

**优点**：第一次立即执行，时效性好。
**缺点**：最后一次事件触发后可能没有执行回调（因为需要等下一次触发且满足时间差条件）。

#### 1.4 节流（定时器版）

```javascript
/**
 * 节流函数（定时器版）
 * 特点：第一次触发延迟执行（等 delay 毫秒后）
 * @param {Function} fn 回调函数
 * @param {number} delay 间隔时间（毫秒）
 * @returns {Function} 节流处理后的函数
 */
function throttleTimer(fn, delay) {
  let timer = null;

  return function (...args) {
    const context = this;

    // 如果当前没有定时器（即不在冷却期），设置定时器
    if (!timer) {
      timer = setTimeout(() => {
        fn.apply(context, args);
        timer = null; // 执行完后冷却期结束
      }, delay);
    }
  };
}
```

**执行流程分析**：第一次触发时 `timer` 为 `null`，设置一个 delay 毫秒后执行的定时器。在定时器未执行的这段时间内，`timer` 不为 `null`，所有触发都被忽略。定时器执行完毕后将 `timer` 重置为 `null`，下一次触发才能再次设置新定时器。

**优点**：保证最后一次事件触发后的回调能在 delay 毫秒后执行。
**缺点**：第一次触发时不会立即执行，有延迟。

#### 1.5 节流（双剑合璧版）

实际项目中最常用的是将两个版本合二为一的节流函数，既保证第一次立即执行，又保证最后一次能执行：

```javascript
/**
 * 增强版节流：时间戳 + 定时器组合
 * 特点：第一次立即执行，最后一次也会执行
 * @param {Function} fn 回调函数
 * @param {number} delay 间隔时间
 * @returns {Function} 节流处理后的函数
 */
function throttle(fn, delay) {
  let timer = null;
  let previous = 0;

  return function (...args) {
    const context = this;
    const now = Date.now();
    // 距离下次执行还剩多少时间
    const remaining = delay - (now - previous);

    if (remaining <= 0) {
      // 超过冷却期，立即执行
      if (timer) {
        clearTimeout(timer);
        timer = null;
      }
      fn.apply(context, args);
      previous = now;
    } else if (!timer) {
      // 在冷却期内，设置定时器保证最后一次能执行
      timer = setTimeout(() => {
        fn.apply(context, args);
        timer = null;
        previous = Date.now();
      }, remaining);
    }
  };
}
```

这个组合版的核心逻辑是：当时间差超过 delay 时立即执行（时间戳版的行为），同时设置一个「补偿定时器」来捕获最后一次事件触发后还未执行的那一次回调。这样既保证了实时性，又保证了完整覆盖。

### 2. call/apply/bind

#### 2.1 手写 call

```javascript
/**
 * 手写 call
 * 原理：将函数作为目标对象的一个临时属性方法调用
 * @param {Object|null|undefined} context this 指向的对象
 * @param  {...any} args 参数列表
 * @returns {any} 函数执行结果
 */
Function.prototype.myCall = function (context, ...args) {
  // 处理 null/undefined：浏览器环境下指向 window（Node 中指向 global）
  // 简单模拟时指向全局对象
  context = context ?? globalThis;

  // 原始值（如 1, 'abc'）需要包装为对象才能添加属性
  // Object() 可以将原始值转为对应的包装对象
  context = Object(context);

  // 使用 Symbol 创建唯一 key，避免覆盖原有属性
  const key = Symbol('call');

  // 将当前函数（this）作为 context 的方法
  context[key] = this;

  // 执行函数，传入参数
  const result = context[key](...args);

  // 删除临时属性（清理工作）
  delete context[key];

  // 返回执行结果
  return result;
};
```

**底层原理分析**：

`call` 的本质是「改变函数执行时的 `this` 指向」。JavaScript 中函数的 `this` 由**调用方式**决定：作为对象的方法调用时 `this` 指向该对象。手写 `call` 就是利用了这个规则——把函数临时挂载到目标对象的属性上，然后通过对象属性的方式调用，这样函数内部的 `this` 就自然而然地指向了目标对象。

使用 `Symbol` 创建唯一键是一个关键细节：如果用普通字符串作属性名，万一 `context` 上已经有一个同名属性，就会被覆盖。`Symbol` 保证每次创建的键都是唯一的。

`Object(context)` 的另一个作用是处理原始值：当 `context` 是 `1` 或 `'hello'` 这样的原始值时，`Object(1)` 返回 `Number` 包装对象，这样才可以在其上添加属性。

#### 2.2 手写 apply

```javascript
/**
 * 手写 apply
 * 与 call 的唯一区别：参数以数组形式传入
 * @param {Object|null|undefined} context this 指向
 * @param {Array} args 参数数组
 * @returns {any} 函数执行结果
 */
Function.prototype.myApply = function (context, args) {
  context = context ?? globalThis;
  context = Object(context);

  const key = Symbol('apply');

  context[key] = this;

  // 关键区别：如果 args 不存在或不是数组，则不传参执行
  let result;
  if (args && Array.isArray(args)) {
    result = context[key](...args);
  } else {
    result = context[key]();
  }

  delete context[key];

  return result;
};
```

`apply` 与 `call` 的实现几乎一模一样，唯一的区别在于参数的传递方式：`call` 是展开的参数列表，`apply` 是数组。这也解释了为什么 `apply` 常用于需要「参数列表长度不确定」的场景，比如 `Math.max.apply(null, arr)`。

#### 2.3 手写 bind（含 new 行为）

`bind` 是三者中最复杂的一个，因为它返回的**新函数**可以被当作普通函数调用，也可以被当作构造函数（使用 `new` 操作符）调用。当使用 `new` 调用时，`this` 绑定不再指向 `bind` 传入的 `context`，而是指向新创建的实例对象。

```javascript
/**
 * 手写 bind（含 new 行为兼容）
 * 原理：
 * 1. 返回一个新函数
 * 2. 新函数可以作为普通函数调用（this 绑定到 context）
 * 3. 新函数可以作为构造函数调用（this 绑定到新创建的实例，忽略 context）
 * 4. 支持柯里化（部分参数传入）
 * @param {Object|null|undefined} context this 指向
 * @param  {...any} bindArgs bind 时传入的部分参数
 * @returns {Function} 绑定 this 后的新函数
 */
Function.prototype.myBind = function (context, ...bindArgs) {
  // 保存原函数的引用
  const originalFn = this;

  // 返回的新函数
  const boundFn = function (...callArgs) {
    // 关键判断：是否通过 new 调用
    // 如果通过 new 调用，this instanceof boundFn 为 true
    // 此时 this 指向新创建的实例，不绑定到 context
    // 否则绑定到 context
    return originalFn.apply(
      this instanceof boundFn ? this : (context ?? globalThis),
      [...bindArgs, ...callArgs]
    );
  };

  // 维护原型链：如果原函数有 prototype，新函数的 prototype 指向原函数的 prototype
  // 这样通过 new boundFn() 创建的实例才能正确继承原函数的 prototype
  if (originalFn.prototype) {
    boundFn.prototype = originalFn.prototype;
  }

  return boundFn;
};
```

**new 绑定的优先级**：

这是理解 `bind` 最精妙之处的关键。JavaScript 中 `this` 绑定的优先级为：
1. **new 绑定**（最高）> 2. **显式绑定**（call/apply/bind）> 3. **隐式绑定**（对象方法调用）> 4. **默认绑定**（独立函数调用）

当通过 `new boundFn()` 调用时，JavaScript 的 `new` 操作符会做四件事：
1. 创建一个新的空对象
2. 将该对象的 `__proto__` 指向构造函数的 `prototype`
3. 将构造函数的 `this` 指向这个新对象
4. 如果构造函数没有返回对象，返回这个新对象

所以在 `boundFn` 内部，`this instanceof boundFn` 为 `true`，此时 `apply` 的第一个参数传入 `this`（新创建的实例），而不是传入的 `context`。这正是 `bind` 返回的函数作为构造函数时 `this` 绑定失效的实现原理。

**原型链的维护**：为了让 `new boundFn()` 创建的实例能访问原函数原型上的方法，需要将 `boundFn.prototype` 指向 `originalFn.prototype`。这里直接赋引用是简写，更严谨的做法是用中间构造函数来隔离，避免修改 `boundFn.prototype` 污染原函数的原型链。

#### 2.4 精确版 bind（原型隔离）

```javascript
// 更严谨的 bind 实现，避免原型污染
Function.prototype.myBindPrecise = function (context, ...bindArgs) {
  const originalFn = this;

  // 中间构造函数，用于原型链隔离
  const ProxyFn = function () {};
  ProxyFn.prototype = originalFn.prototype;

  const boundFn = function (...callArgs) {
    return originalFn.apply(
      this instanceof ProxyFn ? this : (context ?? globalThis),
      [...bindArgs, ...callArgs]
    );
  };

  // 关键：boundFn.prototype 继承自 originalFn.prototype
  // 但修改 boundFn.prototype 不影响 originalFn.prototype
  boundFn.prototype = new ProxyFn();

  return boundFn;
};
```

### 3. Promise 核心实现

手写一个完整的 Promise 实现是最具挑战性但也最有收获的部分。我们基于 Promises/A+ 规范，一步一步构建。

```javascript
/**
 * 自定义 Promise（符合 Promises/A+ 规范核心部分）
 */
class MyPromise {
  // 定义三种状态常量
  static PENDING = 'pending';
  static FULFILLED = 'fulfilled';
  static REJECTED = 'rejected';

  constructor(executor) {
    // 初始状态
    this.state = MyPromise.PENDING;
    // 终值（resolve 的结果）
    this.value = undefined;
    // 拒因（reject 的原因）
    this.reason = undefined;
    // 成功回调队列（支持 then 多次调用）
    this.onFulfilledCallbacks = [];
    // 失败回调队列
    this.onRejectedCallbacks = [];

    // resolve 函数：将 pending → fulfilled
    const resolve = (value) => {
      // 状态一旦变更就不能再变
      if (this.state !== MyPromise.PENDING) return;

      // 如果 value 是 Promise，递归等待其完成
      if (value instanceof MyPromise) {
        value.then(resolve, reject);
        return;
      }

      this.state = MyPromise.FULFILLED;
      this.value = value;

      // 异步执行所有已注册的成功回调
      this.onFulfilledCallbacks.forEach(fn => fn());
    };

    // reject 函数：将 pending → rejected
    const reject = (reason) => {
      if (this.state !== MyPromise.PENDING) return;

      this.state = MyPromise.REJECTED;
      this.reason = reason;

      // 异步执行所有已注册的失败回调
      this.onRejectedCallbacks.forEach(fn => fn());
    };

    // 执行 executor，捕获同步异常
    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  /**
   * then 方法：Promise 的核心
   * 实现了链式调用和值的穿透
   * @param {Function} onFulfilled 成功回调
   * @param {Function} onRejected 失败回调
   * @returns {MyPromise} 新的 Promise，实现链式调用
   */
  then(onFulfilled, onRejected) {
    // 参数可选：如果 onFulfilled 不是函数，使用默认函数（值的穿透）
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value;
    // 如果 onRejected 不是函数，使用默认函数（错误穿透）
    onRejected = typeof onRejected === 'function' ? onRejected : reason => { throw reason; };

    // 返回一个新的 Promise，实现链式调用
    const newPromise = new MyPromise((resolve, reject) => {
      // 封装回调执行逻辑
      const handleCallback = (callback, value) => {
        // 使用微任务模拟异步调用（实际 Promise 是微任务，这里用 setTimeout 模拟）
        setTimeout(() => {
          try {
            const result = callback(value);
            // 如果回调返回值是 Promise，等待它完成
            if (result instanceof MyPromise) {
              result.then(resolve, reject);
            } else {
              // 否则直接 resolve 返回值
              resolve(result);
            }
          } catch (error) {
            // 如果回调抛出异常，reject 新 Promise
            reject(error);
          }
        }, 0);
      };

      if (this.state === MyPromise.FULFILLED) {
        handleCallback(onFulfilled, this.value);
      } else if (this.state === MyPromise.REJECTED) {
        handleCallback(onRejected, this.reason);
      } else {
        // pending 状态：注册回调，等状态变更后再执行
        this.onFulfilledCallbacks.push(() => {
          handleCallback(onFulfilled, this.value);
        });
        this.onRejectedCallbacks.push(() => {
          handleCallback(onRejected, this.reason);
        });
      }
    });

    return newPromise;
  }

  /**
   * catch 方法：语法糖，捕获拒绝状态
   * @param {Function} onRejected 失败回调
   * @returns {MyPromise}
   */
  catch(onRejected) {
    return this.then(null, onRejected);
  }

  /**
   * finally 方法：无论成功失败都会执行
   * @param {Function} callback
   * @returns {MyPromise}
   */
  finally(callback) {
    return this.then(
      value => MyPromise.resolve(callback()).then(() => value),
      reason => MyPromise.resolve(callback()).then(() => { throw reason; })
    );
  }
}
```

**状态机设计的核心细节**：

**状态不可逆**：`resolve` 和 `reject` 的第一行都是 `if (this.state !== PENDING) return;`。这保证了 Promise 的状态一经变更就不可再变，不管后续调用多少次 `resolve` 或 `reject` 都无效。

**回调队列**：因为 `then` 可以在 Promise 状态已经变更后调用（直接执行回调），也可以在 Promise 仍为 `pending` 时调用（将回调存入队列，等待 `resolve`/`reject` 时消费）。使用数组 `onFulfilledCallbacks` 和 `onRejectedCallbacks` 是为了支持多次调用 `then` 的场景。

**链式调用**：每个 `then` 返回一个新的 `MyPromise` 实例。新 Promise 的 resolve/reject 时机取决于回调执行的结果。如果回调返回普通值，新 Promise 立即 resolve 该值；如果返回 Promise，则等待该 Promise 完成后 resolve/reject。这种嵌套 Promise 的等待机制是链式调用的核心。

**值的穿透**：当 `then` 不传参（或传入非函数）时，默认的 `onFulfilled = value => value` 和 `onRejected = reason => { throw reason; }` 保证了值/错误能穿透到链中的下一个 `then`。这就是为什么 `.then().then(v => v)` 能够正常工作。

**setTimeout 模拟异步**：真实的 `then` 回调是在微任务队列中执行的，这里使用 `setTimeout` 来模拟异步行为，保证 `then` 不会在当前同步执行栈中立即执行。

#### 3.1 静态 resolve 方法

```javascript
/**
 * Promise.resolve：返回一个状态为 fulfilled 的 Promise
 * 如果传入值是 Promise，返回它本身
 * 如果传入值是 thenable 对象（有 then 方法），将其转换为 Promise
 * 否则返回值为 resolved 值的 Promise
 */
MyPromise.resolve = function (value) {
  if (value instanceof MyPromise) {
    return value;
  }
  // 处理 thenable
  if (value && typeof value.then === 'function') {
    return new MyPromise((resolve) => value.then(resolve));
  }
  return new MyPromise((resolve) => resolve(value));
};
```

#### 3.2 静态 reject 方法

```javascript
/**
 * Promise.reject：返回一个状态为 rejected 的 Promise
 * 与 resolve 不同：reject 不会对传入值做 thenable 展开
 */
MyPromise.reject = function (reason) {
  return new MyPromise((_, reject) => reject(reason));
};
```

`reject` 与 `resolve` 的一个重要区别是：`reject` 不会对参数做 thenable 展开。如果 `reject` 传入一个 Promise 或 thenable 对象，这个对象本身就是拒绝的原因，不会被展开。

### 4. Promise 静态方法

#### 4.1 手写 Promise.all

```javascript
/**
 * Promise.all：所有 Promise 都成功才成功，一个失败就失败
 * @param {Array} promises Promise 数组
 * @returns {Promise} 新的 Promise
 */
MyPromise.all = function (promises) {
  return new MyPromise((resolve, reject) => {
    // 如果不是数组，直接 reject
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    // 如果空数组，立即 resolve 空数组
    if (promises.length === 0) {
      return resolve([]);
    }

    const results = new Array(promises.length);
    let resolvedCount = 0;

    promises.forEach((promise, index) => {
      // 使用 Promise.resolve 确保非 Promise 值也能被处理
      MyPromise.resolve(promise).then(
        (value) => {
          // 按索引存储结果，保持顺序
          results[index] = value;
          resolvedCount++;

          // 所有 Promise 都 resolve 后，返回结果数组
          if (resolvedCount === promises.length) {
            resolve(results);
          }
        },
        (reason) => {
          // 任何一个 Promise 被 reject，整体 reject
          reject(reason);
        }
      );
    });
  });
};
```

**实现要点**：

1. **保持顺序**：使用 `results[index]` 按索引存储结果，而不是 `push`。这样即使后面的 Promise 先 resolve，结果数组的顺序仍然与输入数组一致。
2. **空数组快速返回**：如果传入空数组，直接 resolve 空数组，不做多余操作。
3. **非 Promise 兼容**：使用 `MyPromise.resolve(promise)` 包装每个元素，这样传入普通值时也能正确处理。
4. **快速失败**：一旦某个 Promise reject，立即 reject 整个 Promise.all，不会等待其他 Promise。

#### 4.2 手写 Promise.race

```javascript
/**
 * Promise.race：谁先完成就用谁的结果（无论成功还是失败）
 * @param {Array} promises Promise 数组
 * @returns {Promise} 新的 Promise
 */
MyPromise.race = function (promises) {
  return new MyPromise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    promises.forEach((promise) => {
      MyPromise.resolve(promise).then(
        (value) => resolve(value), // 第一个 resolve 的胜出
        (reason) => reject(reason) // 第一个 reject 的也胜出
      );
    });
  });
};
```

**实现要点**：

`race` 的精髓在于「竞速」——多个 Promise 任务同时开始执行，谁最先到达终态（无论是 fulfilled 还是 rejected），就用它的结果来决定整个 race 的结果。由于 Promise 状态一经改变就不可逆，resolve 和 reject 函数在被调用过一次之后，后续调用都会被 `if (state !== PENDING) return` 拦截。

#### 4.3 手写 Promise.allSettled

```javascript
/**
 * Promise.allSettled：等待所有 Promise 完成（无论成功或失败）
 * 返回每个 Promise 的结果对象 { status, value/reason }
 * @param {Array} promises Promise 数组
 * @returns {Promise}
 */
MyPromise.allSettled = function (promises) {
  return new MyPromise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    if (promises.length === 0) {
      return resolve([]);
    }

    const results = new Array(promises.length);
    let settledCount = 0;

    promises.forEach((promise, index) => {
      MyPromise.resolve(promise).then(
        (value) => {
          results[index] = { status: 'fulfilled', value };
          settledCount++;
          if (settledCount === promises.length) resolve(results);
        },
        (reason) => {
          results[index] = { status: 'rejected', reason };
          settledCount++;
          if (settledCount === promises.length) resolve(results);
        }
      );
    });
  });
};
```

`allSettled` 与 `all` 的关键区别：`all` 是「一损俱损」，`allSettled` 是「各安天命」。无论成功还是失败，`allSettled` 都会等待所有 Promise 完成，然后用 `{ status, value }` 或 `{ status, reason }` 来描述每个 Promise 的结果。

#### 4.4 手写 Promise.any

```javascript
/**
 * Promise.any：只要有一个成功就成功，全部失败才失败
 * 全部失败时返回 AggregateError
 * @param {Array} promises
 * @returns {Promise}
 */
MyPromise.any = function (promises) {
  return new MyPromise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    const errors = new Array(promises.length);
    let rejectedCount = 0;

    promises.forEach((promise, index) => {
      MyPromise.resolve(promise).then(
        (value) => resolve(value),   // 任意一个成功，整体成功
        (reason) => {
          errors[index] = reason;
          rejectedCount++;
          // 全部失败才 reject
          if (rejectedCount === promises.length) {
            reject(new AggregateError(errors, '所有 Promise 都被拒绝'));
          }
        }
      );
    });
  });
};
```

`any` 与 `race` 的区别需要仔细区分：
- `race`：看谁先到达终态（success or failure），先到者决定结果
- `any`：看谁先成功（只关心 fulfilled），当所有都 rejected 才整体失败

## 实战案例 - 搜索输入框 + 按钮防连点

下面的案例将前面所学的防抖、节流、Promise 串联起来，构建一个带「即时搜索 + 防抖请求 + 提交按钮节流」的实用场景。

```javascript
// ========== 搜索输入框 + 按钮防连点 实战案例 ==========

/**
 * 模拟 API 请求
 * @param {string} keyword 搜索关键词
 * @returns {Promise<string[]>} 搜索结果
 */
function searchAPI(keyword) {
  return new Promise((resolve) => {
    console.log(`📡 发送请求: ${keyword}`);
    setTimeout(() => {
      const results = [`${keyword}的结果1`, `${keyword}的结果2`, `${keyword}的结果3`];
      resolve(results);
    }, 500);
  });
}

/**
 * 用户界面模拟
 */
class SearchUI {
  constructor() {
    this.searchResults = [];
    this.submitLoading = false;

    // 1. 搜索输入防抖：用户停止输入300ms后发送请求
    this.debouncedSearch = debounce(async (keyword) => {
      this.renderLoading(true);
      try {
        const results = await searchAPI(keyword);
        this.searchResults = results;
        this.renderResults();
      } catch (error) {
        console.error('搜索失败:', error);
      } finally {
        this.renderLoading(false);
      }
    }, 300);

    // 2. 提交按钮节流：每 2 秒内至多提交一次
    this.throttledSubmit = throttle(async (formData) => {
      this.submitLoading = true;
      try {
        const result = await this.submitFormAPI(formData);
        console.log('提交成功:', result);
        alert('提交成功！');
      } finally {
        this.submitLoading = false;
      }
    }, 2000);
  }

  /**
   * 用户输入时的回调
   */
  onInput(keyword) {
    if (keyword.trim()) {
      this.debouncedSearch(keyword.trim());
    } else {
      this.searchResults = [];
      this.renderResults();
    }
  }

  /**
   * 用户点击提交按钮
   */
  onSubmit(formData) {
    this.throttledSubmit(formData);
  }

  /**
   * 模拟表单提交
   */
  submitFormAPI(data) {
    return new Promise((resolve) => {
      setTimeout(() => resolve({ success: true, id: Date.now() }), 1000);
    });
  }

  renderLoading(loading) {
    console.log(loading ? '🔄 加载中...' : '✅ 加载完毕');
  }

  renderResults() {
    if (this.searchResults.length === 0) {
      console.log('📭 没有找到结果');
      return;
    }
    console.log('📋 搜索结果:');
    this.searchResults.forEach((result, i) => console.log(`  ${i + 1}. ${result}`));
  }
}

// 使用示例
const ui = new SearchUI();

console.log('=== 搜索场景 ===');
ui.onInput('Java');      // 连续输入
ui.onInput('JavaScript'); // 不会立即请求
setTimeout(() => {
  ui.onInput('JavaScript '); // 再追加一个空格
  // 实际上只会在最后一次输入后 300ms 发送一次请求
}, 100);

console.log('\n=== 提交场景 ===');
ui.onSubmit({ name: 'test' });   // 第一次立即提交
ui.onSubmit({ name: 'test' });   // 2秒内，被节流忽略
ui.onSubmit({ name: 'test' });   // 也被节流忽略
setTimeout(() => {
  ui.onSubmit({ name: 'test2' }); // 2秒后，可以再次提交
}, 2500);
```

在这个实战案例中，防抖和节流各自承担了不同的职责：
- **防抖**应用于搜索输入框，确保只在用户真正停止输入后才请求 API，避免无谓的网络消耗
- **节流**应用于提交按钮，限制提交频率，防止用户在服务器未响应时疯狂点击造成重复提交

一个有趣但很容易混淆的对比：如果搜索框用节流会怎样？用户打字过程中每 300ms 就发一次请求，效果很差。如果提交按钮用防抖会怎样？用户连点 5 次，最后一次点击后等待 2 秒才提交，用户会感觉「卡住了」。所以场景决定策略，没有万能方案。

## 底层原理 - 代码运行机制分析

### 闭包在防抖节流中的作用

防抖和节流的函数中，`timer` 和 `previous` 为什么能被「记住」？核心原因是 JavaScript 的**闭包机制**。

```javascript
function debounce(fn, delay) {
  let timer = null;          // ← 这个变量在闭包中被「捕获」

  return function (...args) {  // ← 匿名函数形成了闭包
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {  // ← 内部函数再次形成闭包
      fn.apply(this, args);
    }, delay);
  };
}
```

当 `debounce` 函数执行完毕后，它的执行上下文被销毁了，但是 `timer` 变量被返回的匿名函数（以及匿名函数内部定义的定时器回调函数）通过作用域链引用着，所以 `timer` 不会被垃圾回收。这就是闭包的本质：**内部函数持有外部函数作用域中变量的引用**。

每次调用防抖函数时，我们操作的 `timer` 始终是第一次调用 `debounce` 时创建的那个变量。所以才能做到「清空之前的定时器，重新设置新定时器」的效果。

### this 绑定规则与优先级

当 `debounce` 返回的函数被调用时，它内部的 `this` 指向谁？

```javascript
const obj = {
  name: 'searchBox',
  onChange: debounce(function(e) {
    console.log(this.name);  // this 指向 obj
  }, 300)
};

obj.onChange(e);  // 通过对象方法调用，this → obj
```

但如果这样调用：

```javascript
const onChange = obj.onChange;
onChange(e);  // 独立函数调用，this → undefined（严格模式）或 globalThis
```

所以手写防抖节流时必须用 `fn.apply(context, args)` 来保留原始的 `this`。`context = this` 被捕获在闭包中，在返回的函数中，`this` 就是调用时的 `this`（通过隐式绑定的规则确定），然后我们用 `apply` 将原函数的 `this` 指向同一对象。

### new 操作符的四个步骤

理解 `bind` 的 `new` 兼容行为，需要深入理解 `new` 操作符的执行流程：

```javascript
/**
 * 手动模拟 new 操作符
 */
function myNew(Constructor, ...args) {
  // 1. 创建一个新的空对象
  const obj = {};

  // 2. 将空对象的原型指向构造函数的 prototype
  Object.setPrototypeOf(obj, Constructor.prototype);
  // 等价于: obj.__proto__ = Constructor.prototype;

  // 3. 将构造函数的 this 指向这个新对象并执行
  const result = Constructor.apply(obj, args);

  // 4. 如果构造函数返回了一个对象，返回该对象；否则返回新创建的对象
  return (typeof result === 'object' && result !== null) || typeof result === 'function'
    ? result
    : obj;
}
```

当 `new boundFn()` 调用 `myBind` 返回的函数时，JavaScript 引擎内部自动完成了上述四个步骤。在 `boundFn` 内部，`this instanceof boundFn` 检查的是：`this.__proto__`（即新对象的原型）是否等于 `boundFn.prototype`。因为我们在 `myBind` 中将 `boundFn.prototype` 指向了 `originalFn.prototype`，所以新创建的对象能够正确继承原函数的原型链。

这就是为什么 `bind` 返回的函数可以作为构造函数使用——因为它返回的 `boundFn` 的 `prototype` 被手动链接到了原函数的 `prototype` 上。

### 微任务与宏任务

真实的 Promise 使用**微任务（microtask）**来调度 `then` 回调，这意味着 `then` 回调会在当前宏任务执行完毕后、下一个宏任务执行前被调用。而我们手写的 `MyPromise` 使用 `setTimeout`（宏任务）来模拟异步，存在精度差异：

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0); // 宏任务
Promise.resolve().then(() => console.log('3')); // 微任务
console.log('4');

// 输出顺序: 1 → 4 → 3 → 2
// 使用 setTimeout 模拟的 MyPromise: 1 → 4 → 2 → 3
```

真实的微任务（`queueMicrotask`、`MutationObserver`、`process.nextTick`）总是先于宏任务执行。但在面试中，面试官更关注的是「状态机」「链式调用」「值的穿透」等核心逻辑的实现，而不是微任务调度（后者用 `queueMicrotask` 可以轻松修正）。

使用 `queueMicrotask` 替换 `setTimeout` 可以获得真实的微任务行为：

```javascript
const handleCallback = (callback, value) => {
  // 使用真实的微任务
  queueMicrotask(() => {
    try {
      const result = callback(value);
      if (result instanceof MyPromise) {
        result.then(resolve, reject);
      } else {
        resolve(result);
      }
    } catch (error) {
      reject(error);
    }
  });
};
```

## 高频面试题解析

### 防抖节流篇

**Q1: 防抖和节流的区别是什么？分别适合什么场景？**

A: 防抖是「合并触发」，将连续多次触发合并为一次，适合需要「等用户消停后再处理」的场景（搜索输入框、窗口 resize 完成后再计算布局）。节流是「稀释频率」，保证一段时间内至少执行一次，适合「持续触发但需要定期反馈」的场景（滚动加载更多、拖拽元素的坐标更新）。

记忆口诀：**防抖像电梯等人一起走，节流像地铁到点就发车。**

**Q2: 防抖的 immediate 参数什么时候用？**

A: 当希望第一次点击就立即生效，但后续连续点击被忽略时使用。典型场景是提交按钮——用户点第一次就立即提交，如果在冷却期内又点了多次，只生效第一次。另一种场景是点赞按钮的乐观更新，第一次点击立即改变 UI 状态，然后合并后续的连击。

**Q3: 节流的时间戳版和定时器版有什么区别？**

A: 时间戳版第一次触发立即执行但最后一次可能被吞掉；定时器版第一次触发延迟执行但保证最后一次能执行。实际开发中通常将两者组合使用。

### call/apply/bind 篇

**Q4: call 和 apply 的区别仅仅是参数形式不同吗？性能上有差异吗？**

A: 主要区别确实只是参数形式。性能上，在 V8 引擎中，`call` 通常比 `apply` 快一点（因为 `apply` 需要对参数数组做额外的解构操作），但在日常开发中可以忽略不计。一个非标准的优化技巧是：当参数数量不超过 3 个时，`call` 更快；参数较多时 `apply` 更方便。

**Q5: bind 的 this 绑定能被 call 或 apply 改变吗？**

A: 不能。`bind` 返回的函数有一个「硬绑定」特性——`bind` 内部通过 `apply` 指定了 `this`，而这个 `apply` 是在 `boundFn` 内部执行的，后续再对这个 `boundFn` 调用 `call` 或 `apply`，并不会影响 `boundFn` 内部 `apply` 的 `this` 参数。但有一个例外：通过 `new` 调用 `boundFn` 时，`this` 绑定为 `new` 创建的实例，忽略 `bind` 绑定的 `this`。

**Q6: 一个函数可以被 bind 多次，最终 this 指向什么？**

A: 指向第一次 bind 的目标。`bind` 返回的新函数一旦绑定，再次对其调用 `bind` 是在新函数上绑定，不会影响原函数的绑定。这也是硬绑定（hard binding）的含义。

### Promise 篇

**Q7: Promise 的状态为什么不可逆？**

A: 这是 Promise 设计原则中的核心约定。状态不可逆（pending → fulfilled/rejected）保证了 `then` 注册的回调只会被调用恰好一次，不会出现先 resolve 后 reject 的矛盾状态。这种设计让 Promise 的结果具有确定性，是异步流程可预测的基础。

**Q8: then 链式调用中，一个 then 返回了一个 Promise，下一个 then 是如何拿到这个 Promise 内部的值的？**

A: 这是通过「递归展开」实现的。当 `then` 的回调返回一个 Promise 时，手写的实现中会调用 `result.then(resolve, reject)` ——即等待这个返回的 Promise 到达终态，然后再去 resolve（或 reject）当前 then 返回的新 Promise。这样递归向下展开，直到某个 then 返回普通值为止。这就是所谓的「Promise 展开」。

**Q9: Promise.all 和 Promise.allSettled 的区别是什么？什么时候用哪个？**

A: `all` 只要有一个 Promise 被 reject 就整体 reject，适合「所有任务必须全部成功」的场景（如上传多个文件，任何一个失败都应该中断）。`allSettled` 等待所有 Promise 完成（无论成功还是失败），适合「需要知道每个任务的结果」的场景（如批量数据导入，想记录哪些成功哪些失败，不因个别失败而放弃整个批次）。

**Q10: Promise.race 有个什么常见的「坑」？**

A: 一个常见误解是认为 `race` 获胜的 Promise 完成后，其他 Promise 就不会执行了。实际上，被 `race` 抛弃的 Promise 仍然会执行到底，只是其结果被忽略了。这可能导致「幽灵请求」——发出去的请求仍然在消耗网络资源，但代码已经不再关心它的结果了。使用 `AbortController` 可以主动取消被抛弃的请求。

```javascript
// race 的幽灵请求问题
Promise.race([
  fetch('/api/fast'),
  fetch('/api/slow') // 这个请求即使 race 失败了，仍然会发出去
]);

// 使用 AbortController 主动取消
const controller = new AbortController();
setTimeout(() => controller.abort(), 3000); // 3秒后超时取消
fetch('/api/slow-data', { signal: controller.signal });
```

## 总结与扩展

本文从防抖、节流、call/apply/bind、Promise 这四大核心手写题入手，逐层拆解了每个函数的实现原理和底层运行机制。回顾这些知识点的核心脉络：

**防抖与节流**体现了「不同策略解决相同问题」的工程思想。它们的共同目标是限制高频回调的执行频率，但实现方式完全不同——防抖靠「重置计时器」来推迟执行，节流靠「时间比较+定时器门闩」来控制执行间隔。

**call/apply/bind** 揭示了 JavaScript 中 `this` 绑定的底层规则。通过手写这三个函数，我们触摸到了函数调用的底层机制：函数是如何通过 `this` 找到执行上下文的，`new` 操作的四个步骤是如何影响 `this` 指向的，原型链又是如何在构造函数之间传递的。

**Promise** 是所有这些知识点中最深刻也最复杂的一个。完整实现一个 Promise 需要掌握：状态机设计、异步回调队列调度、链式调用的递归展开、异常处理传导、值的穿透机制。每一个细节都对应着一个 JavaScript 语言的核心概念。

### 扩展方向

如果上述手写题已经熟练掌握了，可以进一步挑战：

1. **async/await 的实现原理**：本质上是一个 generator + 自动执行器（如 `co` 库）的语法糖。可以尝试手写一个 async 函数的 polyfill。

2. **事件循环（Event Loop）的深度理解**：宏任务（setTimeout/setInterval/I/O）与微任务（Promise.then/queueMicrotask/process.nextTick）的执行顺序。可以用节点版本中的 `node --print` 验证。

3. **AbortSignal + Promise 的可取消方案**：如何实现一个可取消的 Promise？AbortController 的使用与 promise 的结合。

4. **并发控制**：手写一个限制最大并发数的 `Promise.allLimit`（比如 `p-limit` 库的核心实现），控制同时发送的请求数量。

5. **debounce 和 throttle 的 TypeScript 版本**：为这些函数加上完善的泛型类型推导，让用户在使用时有完美的类型提示。

面试中的手写题从来不是为了难倒你，而是为了在短短几十分钟内最大程度地窥见你的技术广度与深度。每个 `setTimeout`、每个 `apply`、每个 `throw` 背后，都藏着一个关于 JavaScript 运行机制的故事。读懂这些故事，才能真正掌握这门语言。

---

*本文所有手写代码已包含完整注释和可运行示例，读者可以直接复制到浏览器控制台或 Node.js 环境中执行验证。*
