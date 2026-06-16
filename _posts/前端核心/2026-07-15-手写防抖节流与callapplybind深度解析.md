---
layout: post
title: "手写防抖节流与call/apply/bind深度解析：让面试官眼前一亮的完整实现"
date: 2026-07-15 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [防抖, 节流, call, apply, bind, 手写题]
math: true
mermaid: true
image:
  path: /images/default-banner.png
  alt: "手写防抖节流与call/apply/bind深度解析"
---

## 一句话概括

防抖（Debounce）与节流（Throttle）通过精巧的定时器策略控制函数执行频率，而 call/apply/bind 则是 JavaScript 中手动绑定 `this` 的核心机制；理解这些手写实现的完整面（包括各种边界条件、隐式绑定退化、new 调用覆盖）是区分"会用"和"真懂"的分水岭。

## 背景与意义

### 防抖与节流的日常困境

每天早上打开 IDE，你都会遇到这些场景：

```javascript
// 场景 1：搜索输入框
searchInput.addEventListener('input', (e) => {
  // 用户每按一个键就发一次请求 → 后端要被你打死了
  fetchSearchResults(e.target.value);
});

// 场景 2：resize 事件监听
window.addEventListener('resize', () => {
  // 用户拖一下窗口，这个回调触发了 50 次
  recalculateLayout();
});

// 场景 3：滚动加载
window.addEventListener('scroll', () => {
  // 一次滚动触发了 200 次回调
  checkInfiniteScroll();
});
```

防抖和节流就是为解决这类"事件风暴"而生的。虽然 Lodash 提供了现成的 `_.debounce` 和 `_.throttle`，但面试手写它们时，必须考虑：

- **this 指向**：包装后函数的 this 应正确传递
- **参数传递**：事件参数（如 event 对象）必须透传
- **支持取消**：`.cancel()` 方法取消未执行的调用
- **立即执行选项**：`leading` 和 `trailing` 控制边界行为
- **返回值**：`debounce` 的返回值如何给调用方

### call/apply/bind 的深层理解

面试中手写 call/apply/bind，90% 的候选人只能给出"`fn.call(obj, args)` 将 this 指向 obj"这个层面的答案。但考官真正想看的是：

1. 如何用一个 Symbol 属性实现临时 this 绑定
2. apply 的展开参数如何用 ES3/ES6 方式实现
3. bind 返回的函数作为构造函数（`new`）时如何优先绑定
4. 兼容基本类型（string/number/boolean）作为 this 时如何装箱

## 概念与定义

### 防抖（Debounce）

> 在事件被触发 n 秒后执行回调，如果在这 n 秒内又被触发，则重新计时。

**视觉类比**：电梯门。有人进来就延迟关门时间，直到没有人继续进入才真的关门。

**适用场景**：
- 搜索输入实时查询（用户停止输入后再请求）
- 表单验证（用户停手后再校验）
- 窗口尺寸变化完成后重新计算
- 自动保存草稿

### 节流（Throttle）

> 规定一个单位时间，在这个单位时间内只能触发一次回调。如果在这个单位时间内触发多次，只有一次生效。

**视觉类比**：水龙头阀门。无论你怎么快速开关，水流出的速率是固定的。

**适用场景**：
- 滚动加载（限制触发频率，不能用户每滚一像素就发请求）
- 拖拽、resize 频繁触发的事件
- 游戏中的技能冷却
- 射击频率控制

### call/apply/bind

| 方法 | 功能 | 执行时机 | 参数形式 |
|------|------|----------|----------|
| `call(thisArg, arg1, arg2, ...)` | 调用函数并指定 this | 立即执行 | 参数列表 |
| `apply(thisArg, [argsArray])` | 调用函数并指定 this | 立即执行 | 参数数组 |
| `bind(thisArg, arg1, arg2, ...)` | 返回绑定 this 的新函数 | 返回新函数，不立即执行 | 参数列表（可部分应用） |

## 最小示例

### 防抖最小实现

```javascript
/**
 * @param {Function} fn - 需要防抖的函数
 * @param {number} delay - 延迟毫秒数
 * @returns {Function} 防抖处理后的函数
 */
function debounce(fn, delay = 300) {
  let timer = null;

  return function (...args) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, args);
      timer = null;
    }, delay);
  };
}
```

### 节流最小实现（时间戳版）

```javascript
/**
 * @param {Function} fn - 需要节流的函数
 * @param {number} interval - 时间间隔（ms）
 * @returns {Function} 节流处理后的函数
 */
function throttle(fn, interval = 200) {
  let lastTime = 0;

  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}
```

### call 最小实现

```javascript
Function.prototype.myCall = function (context, ...args) {
  // 如果 context 为 null/undefined，指向全局对象（浏览器中是 window）
  context = context ?? globalThis;
  // 基本类型要装箱
  if (typeof context !== 'object' && typeof context !== 'function') {
    context = Object(context);
  }

  const key = Symbol('temp');
  context[key] = this;
  const result = context[key](...args);
  delete context[key];
  return result;
};
```

### apply 最小实现

```javascript
Function.prototype.myApply = function (context, argsArray) {
  context = context ?? globalThis;
  if (typeof context !== 'object' && typeof context !== 'function') {
    context = Object(context);
  }

  const key = Symbol('temp');
  context[key] = this;
  const result = argsArray
    ? context[key](...argsArray)
    : context[key]();
  delete context[key];
  return result;
};
```

### bind 最小实现

```javascript
Function.prototype.myBind = function (context, ...bindArgs) {
  const originalFn = this;

  return function (...callArgs) {
    // 注意：bind 返回的函数不能处理 new 调用
    return originalFn.apply(context, [...bindArgs, ...callArgs]);
  };
};
```

## 核心知识点拆解

### 1. 防抖的完整形态：leading & trailing & cancel

完整的 `debounce` 实现需要考虑三个执行模式：

- **leading**（立即执行）：触发时立即执行一次，然后等待停止触发才执行 trailing
- **trailing**（停止后执行）：默认模式，触发结束后延迟执行
- **两者同时为 true**：触发时立即执行，停止触发后延迟执行（如失焦提交的双保险）

```javascript
/**
 * 完整的防抖实现（含 leading/trailing 和 cancel/ flush）
 */
function debounce(fn, delay = 300, options = {}) {
  const { leading = false, trailing = true, maxWait } = options;

  if (!trailing && !leading) {
    throw new Error('At least one of leading/trailing must be true');
  }

  let timer = null;
  let lastArgs = null;
  let lastThis = null;
  let lastCallTime = null;
  let lastInvokeTime = 0;

  // 计算离下一次调用的等待时间
  function remainingWait(time) {
    const timeSinceLastCall = time - lastCallTime;
    const timeSinceLastInvoke = time - lastInvokeTime;
    const timeWaiting = delay - timeSinceLastCall;

    // 如果设置了 maxWait，使用 maxWait 限制
    return maxWait
      ? Math.min(timeWaiting, maxWait - timeSinceLastInvoke)
      : timeWaiting;
  }

  function invokeFunc(time) {
    lastInvokeTime = time;
    timer = null;
    if (lastArgs) {
      fn.apply(lastThis, lastArgs);
    }
    lastArgs = lastThis = null;
  }

  function leadingEdge(time) {
    lastInvokeTime = time;
    // 立即执行
    if (leading) {
      invokeFunc(time);
    }
    // 启动后续的 trailing 等待
    timer = setTimeout(() => timerExpired(time), delay);
  }

  function timerExpired(time) {
    const wait = remainingWait(time);
    if (wait <= 0) {
      // 是时候执行 trailing 了
      if (trailing && lastArgs) {
        invokeFunc(time);
      }
    } else {
      // 这不可能发生（setTimeout 不会提前触发），但安全处理
      timer = setTimeout(() => timerExpired(time), wait);
    }
  }

  function debounced(...args) {
    const time = Date.now();
    const isInvoking = shouldInvoke(time);

    lastArgs = args;
    lastThis = this;
    lastCallTime = time;

    if (isInvoking) {
      if (timer === null) {
        leadingEdge(time);
      } else if (maxWait) {
        // 已到达 maxWait 上限，强制执行
        timerExpired(time);
      }
    }
  }

  function shouldInvoke(time) {
    const timeSinceLastCall = time - lastCallTime;
    const timeSinceLastInvoke = time - lastInvokeTime;

    return (
      lastCallTime === null
      || timeSinceLastCall >= delay
      || timeSinceLastCall < 0          // 系统时间被修改了
      || (maxWait && timeSinceLastInvoke >= maxWait)
    );
  }

  // 取消：取消待执行的调用
  debounced.cancel = function () {
    if (timer !== null) {
      clearTimeout(timer);
      timer = null;
    }
    lastArgs = lastThis = lastCallTime = null;
  };

  // 立即执行未决调用
  debounced.flush = function () {
    if (timer !== null && lastArgs) {
      invokeFunc(Date.now());
    }
  };

  return debounced;
}
```

### 2. 节流的完整形态：leading + trailing

类似地，`throttle` 也有 leading 和 trailing 两个边界：

```javascript
function throttle(fn, interval = 300, options = {}) {
  const { leading = true, trailing = true } = options;

  let timer = null;
  let lastArgs = null;
  let lastThis = null;
  let previous = 0;

  function invokeFunc(time) {
    previous = time;
    timer = null;
    fn.apply(lastThis, lastArgs);
    lastArgs = lastThis = null;
  }

  function throttled(...args) {
    const now = Date.now();
    // 首次调用：如果 leading 为 false，则不执行，仅记录时间
    if (!previous && leading === false) {
      previous = now;
    }

    const remaining = interval - (now - previous);
    lastArgs = args;
    lastThis = this;

    if (remaining <= 0 || remaining > interval) {
      // 到达执行时机
      if (timer) {
        clearTimeout(timer);
        timer = null;
      }
      invokeFunc(now);
    } else if (!timer && trailing !== false) {
      // 设置 trailing 执行
      timer = setTimeout(() => {
        invokeFunc(Date.now());
      }, remaining);
    }
  }

  throttled.cancel = function () {
    if (timer) {
      clearTimeout(timer);
      timer = null;
    }
    previous = 0;
    lastArgs = lastThis = null;
  };

  return throttled;
}
```

### 3. 防抖 vs 节流：什么时候用哪个？

| 对比维度 | 防抖 (Debounce) | 节流 (Throttle) |
|----------|-----------------|-----------------|
| 执行频率 | 停手后执行一次 | 固定频率执行 |
| 延迟特征 | 可无限推迟 | 最多推迟一个周期 |
| 适合场景 | 搜索、验证、自动保存 | 滚动、resize、动画帧 |
| 第一次触发 | 可选立即（leading） | 默认立即（可配置） |
| 最后一次触发 | 总是执行（trailing） | 可选（trailing） |

**判断标准**：你需要的是"结果"还是"过程"？
- 搜索要的是"最终结果"→ 防抖
- 滚动要的是"过程中的反馈"→ 节流

### 4. 手写 call：Symbol 属性的陷阱

使用 Symbol 作为临时属性可以避免覆盖 context 上已有的属性名。但还有更多细节：

```javascript
// 完整的 call 实现教学
Function.prototype.myCall = function (context, ...args) {
  // Step 1: 处理 null/undefined → 指向全局对象
  // 在严格模式下 this 是 undefined，在非严格模式下是 window/global
  context = context ?? globalThis;

  // Step 2: 基本类型装箱
  // 如果传的是 42 或 'hello' 等基本类型，要转为对象
  const contextType = typeof context;
  if (contextType !== 'object' && contextType !== 'function') {
    context = Object(context);
  }

  // Step 3: 用 Symbol 创建唯一 key，避免属性覆盖
  const fnKey = Symbol('fn');

  // Step 4: 将调用函数设为 context 的方法
  context[fnKey] = this;

  // Step 5: 调用并获取返回值
  // 使用 Reflect 可以更安全地调用
  const result = context[fnKey](...args);

  // Step 6: 清理临时属性
  delete context[fnKey];

  return result;
};

// 测试
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const person = { name: 'Alice' };
console.log(greet.myCall(person, 'Hello', '!')); // Hello, Alice!

// 边界测试
console.log(greet.myCall(null, 'Hi', '?')); // Hi, undefined? (window.name === '')
console.log(greet.myCall(42, 'Hey', '.'));  // Hey, undefined. (Number对象无name)
```

### 5. 手写 apply：与 call 的唯一区别是参数形式

```javascript
Function.prototype.myApply = function (context, argsArray) {
  context = context ?? globalThis;
  const contextType = typeof context;
  if (contextType !== 'object' && contextType !== 'function') {
    context = Object(context);
  }

  const fnKey = Symbol('fn');
  context[fnKey] = this;

  let result;
  if (argsArray === null || argsArray === undefined) {
    // 未传参数列表时，不传参调用
    result = context[fnKey]();
  } else if (!Array.isArray(argsArray)) {
    // 规范要求必须传类数组或 undefined，但实际面试中可能不传
    throw new TypeError('CreateListFromArrayLike called on non-object');
  } else {
    result = context[fnKey](...argsArray);
  }

  delete context[fnKey];
  return result;
};
```

### 6. 手写 bind：最复杂的 this 绑定

`bind` 的核心难点在于：**当返回的函数被 `new` 调用时，绑定的 `this` 应被覆盖**。

```javascript
Function.prototype.myBind = function (context, ...bindArgs) {
  if (typeof this !== 'function') {
    throw new TypeError('myBind must be called on a function');
  }

  const originalFn = this;

  // 创建中间函数，用于原型链连接
  const fNOP = function () {};
  // 用 fBound 作为 bind 返回的函数
  const fBound = function (...callArgs) {
    // 当作为构造函数调用时：
    //   this 指向新创建的实例
    //   fBound.prototype 是 new 出来的对象的原型
    // 判断依据：this instanceof fNOP 为 true
    return originalFn.apply(
      this instanceof fNOP ? this : context,
      [...bindArgs, ...callArgs]
    );
  };

  // 维护原型链
  // 如果 originalFn 有 prototype，则 fBound 的 prototype 继承它
  if (originalFn.prototype) {
    fNOP.prototype = originalFn.prototype;
  }
  fBound.prototype = new fNOP();

  return fBound;
};

// 测试
function Person(name, age) {
  this.name = name;
  this.age = age;
}

const BoundPerson = Person.myBind({}, 'Alice');
const alice = new BoundPerson(25);
console.log(alice.name);  // Alice（正确：new 覆盖了绑定的 {}）
console.log(alice.age);   // 25
console.log(alice instanceof Person); // true（原型链正确）

const boundGreet = greet.myBind({ name: 'Bob' });
console.log(boundGreet('Hola', '?')); // Hola, Bob?
```

### 7. 一个特殊问题：箭头函数与 this

箭头函数没有自己的 `this`，不能被 `call/apply/bind` 改变 this 指向：

```javascript
const arrowFn = () => this.name;

const obj = { name: 'objName' };
console.log(arrowFn.call(obj));  // undefined（箭头函数的 this 不能被改变）

// 手写 call 中必须处理这种情况？
// 不需要——因为箭头函数根本没有 [[BoundThis]] 的概念
// 调用 call 时，即使我们通过 context[fnKey]() 调用，箭头函数的 this 仍然
// 取创建时外层作用域的 this，而不是 context

function regularFn() {
  return this.name;
}
console.log(regularFn.call(obj)); // 'objName'
```

这一区别是 JavaScript 语言设计的核心特性，手写 `call` 不需要额外处理——因为箭头函数在底层就直接不使用传入的 this 值。

## 实战案例：实时搜索与内容无限滚动

### 实时搜索（防抖 + 取消）

```javascript
class SearchComponent {
  constructor(inputElement, resultContainer) {
    this.input = inputElement;
    this.results = resultContainer;
    this.abortController = null;

    this.setupDebouncedSearch();
  }

  setupDebouncedSearch() {
    this.debouncedSearch = debounce(
      (query) => this.performSearch(query),
      400,
      { leading: false, trailing: true }
    );

    this.input.addEventListener('input', (e) => {
      // 每次输入时取消上一次未完成的请求
      if (this.abortController) {
        this.abortController.abort();
      }
      this.debouncedSearch(e.target.value);
    });
  }

  async performSearch(query) {
    if (!query || query.length < 2) {
      this.results.innerHTML = '';
      return;
    }

    this.abortController = new AbortController();

    try {
      this.results.innerHTML = '<div class="loading">搜索中...</div>';

      const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
        signal: this.abortController.signal,
      });

      if (!response.ok) throw new Error('Search failed');

      const data = await response.json();
      this.renderResults(data);
    } catch (err) {
      if (err.name !== 'AbortError') {
        this.results.innerHTML = '<div class="error">搜索出错了</div>';
        console.error('Search error:', err);
      }
      // AbortError 是正常的：用户又输入了新内容，不需要显示错误
    }
  }

  renderResults(data) {
    if (data.length === 0) {
      this.results.innerHTML = '<div class="no-results">未找到相关结果</div>';
      return;
    }

    this.results.innerHTML = data.map(item => `
      <div class="result-item" data-id="${item.id}">
        <h4>${this.highlight(item.title)}</h4>
        <p>${item.description}</p>
      </div>
    `).join('');
  }

  highlight(text) {
    // 高亮匹配文本（示例简化）
    return text;
  }
}
```

### 无限滚动加载（节流）

```javascript
class InfiniteScroll {
  constructor(container, loadMoreFn) {
    this.container = container;
    this.loadMore = loadMoreFn;
    this.page = 1;
    this.loading = false;
    this.hasMore = true;

    this.throttledScroll = throttle(
      () => this.onScroll(),
      200,
      { leading: true, trailing: true }
    );

    this.container.addEventListener('scroll', this.throttledScroll, {
      passive: true,
    });
  }

  onScroll() {
    if (this.loading || !this.hasMore) return;

    const { scrollTop, scrollHeight, clientHeight } = this.container;

    // 距离底部 200px 时触发加载
    if (scrollHeight - scrollTop - clientHeight < 200) {
      this.loadNextPage();
    }
  }

  async loadNextPage() {
    this.loading = true;
    this.page++;

    // 显示加载指示器
    const loader = this.showLoader();

    try {
      const data = await this.loadMore(this.page);

      if (data.length === 0) {
        this.hasMore = false;
        this.showEndMessage();
      } else {
        this.renderItems(data);
      }
    } catch (err) {
      this.page--; // 失败回滚页码
      console.error('加载失败:', err);
      this.showError();
    } finally {
      loader.remove();
      this.loading = false;
    }
  }
}
```

## 底层原理

### 1. V8 中 Function.prototype.bind 的底层实现

在 V8 引擎中，`Function.prototype.bind` 的实现位于 `src/builtins/builtins-function.cc`。其核心逻辑如下（伪代码）：

```c++
// V8 bind 简化逻辑
Object FunctionBind(Function target, Object thisArg, Array args) {
  // 创建 BoundFunction 实例
  BoundFunction bound = new BoundFunction();
  bound.bound_target_function = target;
  bound.bound_this = thisArg;
  bound.bound_arguments = args;

  // 处理 length 属性：原始函数 length - 已绑定参数个数
  int targetLength = target.length;
  int boundLength = max(0, targetLength - args.length);
  bound.length = boundLength;

  // 设置 name 属性：'bound ' + 原始函数名
  bound.name = "bound " + target.name;

  // 处理原型
  if (target.hasInstancePrototype()) {
    bound.prototype = new Object();
    bound.prototype.__proto__ = target.prototype;
  } else {
    bound.prototype = undefined;
  }

  return bound;
}
```

关键点：
- bind 返回的是一个 `BoundFunction` 类型的对象（不同于普通函数）
- `BoundFunction` 的 [[Call]] 和 [[Construct]] 两种调用模式分别实现
- [[Construct]] 模式下：忽略绑定的 `thisArg`，使用新创建的 `this`

```c++
// [[Call]] 调用
Object CallBoundFunction(BoundFunction bound, Object thisArg, Array callArgs) {
  // 合并参数
  Array allArgs = bound.bound_arguments + callArgs;
  // [[Call]] 模式下使用绑定的 thisArg
  return Call(bound.bound_target_function, bound.bound_this, allArgs);
}

// [[Construct]] 调用（new 操作符）
Object ConstructBoundFunction(BoundFunction bound, Array args) {
  Array allArgs = bound.bound_arguments + args;
  // [[Construct]] 模式下忽略 bound_this，target 自己创建 this
  return Construct(bound.bound_target_function, allArgs);
}
```

### 2. 原始 this 指向规则复习

JavaScript 规范中的 this 绑定优先级（从高到低）：

1. **new 绑定**：`new fn()` — this 指向新创建的对象
2. **显式绑定**：`fn.call(obj)` / `fn.apply(obj)` / `fn.bind(obj)()`
3. **隐式绑定**：`obj.fn()` — this 指向 obj
4. **默认绑定**：独立函数调用 `fn()` — 非严格模式下指向 globalThis，严格模式下为 undefined

**手写 call/apply/bind 的核心**：就是通过"隐式绑定"的方式实现"显式绑定"的效果——将函数赋值给 context 的一个临时属性，然后通过 `context.fn()` 来调用。

### 3. setTimeout 的最小延迟

防抖和节流中使用 `setTimeout` 时，有一个重要限制：**浏览器 HTML 规范规定 `setTimeout` 的最小延迟为 4ms**（嵌套调用超过 5 层时为 4ms，未嵌套时至少 0ms 但实际至少 1ms）。

这意味着：

```javascript
// 即使 delay 设为 0，实际延迟至少 ~1ms
debounce(fn, 0); // 不会立刻执行，至少等待 ~1ms

// 在后台标签页中，Chrome 会将 setTimeout 最小间隔限制为 1000ms！
// 这意味着后台 tab 的防抖可能严重延迟
```

**对防抖/节流的影响**：如果你的 delay 设置非常小（如 `debounce(fn, 5)`），实际效果可能不如预期，因为浏览器最小延迟会导致实际的防抖间隔大于 5ms。

## 高频面试题解析

### 面试题 1：手写防抖时，为什么 `fn.apply(this, args)` 中的 this 用箭头函数不生效？

**解答**：

防抖函数返回的包装函数中，`this` 指向调用时的上下文。如果使用箭头函数：

```javascript
function debounce(fn, delay) {
  let timer = null;
  return (...args) => {
    // ❌ 箭头函数的 this 取自上层作用域（此处是 debounce 的 this，即 undefined 或 window）
    // 而不是调用时的上下文
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
    //                                       ^^^^ 这里的 this 错了
  };
}

// 使用
button.addEventListener('click', debounce(function() {
  console.log(this); // 应该是 button，但 get 到的是 window
}, 300));
```

解决方案：使用普通函数，这样 `this` 正确绑定到调用上下文（button）。

### 面试题 2：手写 bind 的 new 调用优先级判断

问：为什么 `this instanceof fNOP` 可以判断是否被 `new` 调用？

**解答**：

```javascript
Function.prototype.myBind = function (context, ...bindArgs) {
  const originalFn = this;
  const fNOP = function () {};

  const fBound = function (...callArgs) {
    // 如何判断 fBound 是否被 new 调用？
    return originalFn.apply(
      this instanceof fNOP ? this : context,
      [...bindArgs, ...callArgs]
    );
  };

  fNOP.prototype = originalFn.prototype;
  fBound.prototype = new fNOP();

  return fBound;
};
```

当使用 `new fBound()` 时，JavaScript 引擎会：
1. 创建一个新对象 `obj`
2. 将 `obj.__proto__` 指向 `fBound.prototype`
3. 执行 `fBound.call(obj, ...args)`

在 `fBound` 内部，`this` 指向那个新对象 `obj`。因为 `fBound.prototype` 是通过 `new fNOP()` 创建的，它的原型链是：

```
obj → fBound.prototype → fNOP.prototype → originalFn.prototype → ...
```

`obj instanceof fNOP` 会沿着原型链查找 `fNOP.prototype`，确实找到了，所以返回 `true`。因此 `this instanceof fNOP` 成立意味着被 `new` 调用。

当普通调用 `fBound()` 时，`this` 指向调用的上下文（可能是 window 或 undefined），它不会继承 `fNOP.prototype`，因此 `this instanceof fNOP` 为 `false`，就会使用最初绑定的 `context`。

### 面试题 3：有一个场景——搜索框需要即刻响应用户输入（实时搜索建议），但又不能请求太频繁。同时用户停止输入后还要发出最后一个请求（防抖）。但每次请求前需要先取消上一次未完成的请求。要求同时实现防抖、请求取消、和 Loading 状态管理，写出完整代码。

**解答**：

这是一个综合了防抖、AbortController 和状态管理的实际问题：

```javascript
class SmartSearch {
  constructor(inputEl, options = {}) {
    this.input = inputEl;
    this.debounceDelay = options.debounceDelay || 300;
    this.minQueryLength = options.minQueryLength || 2;
    this.apiEndpoint = options.apiEndpoint || '/api/search';

    this.autoAbortController = null;
    this.manualAbortController = null;
    this.lastQuery = '';
    this.state = 'idle'; // idle | loading | success | error

    // 创建包含 leading 的防抖，让第一次输入也有即时反馈
    this.search = debounce(
      (query) => this.executeSearch(query),
      this.debounceDelay,
      { leading: true, trailing: true }
    );

    this.input.addEventListener('input', (e) => this.handleInput(e));
  }

  handleInput(e) {
    const query = e.target.value.trim();

    if (query === this.lastQuery) return;
    this.lastQuery = query;

    if (query.length < this.minQueryLength) {
      this.clearSuggestions();
      this.emitState('idle');
      return;
    }

    // 取消上一次正在执行的请求
    if (this.autoAbortController) {
      this.autoAbortController.abort();
    }

    this.search(query);
  }

  async executeSearch(query) {
    // 如果 query 已被后续覆盖，不执行
    if (query !== this.lastQuery) return;

    this.autoAbortController = new AbortController();
    const signal = this.autoAbortController.signal;
    this.emitState('loading');

    try {
      const response = await fetch(
        `${this.apiEndpoint}?q=${encodeURIComponent(query)}`,
        { signal }
      );

      if (!response.ok) throw new Error(`HTTP ${response.status}`);

      const results = await response.json();

      // 再次检查 query 是否已被覆盖（异步操作之间可能变化）
      if (query !== this.lastQuery) return;

      this.renderSuggestions(results);
      this.emitState('success', results);
    } catch (err) {
      if (err.name === 'AbortError') {
        // 正常取消，不需要处理
        this.emitState('idle');
        return;
      }
      this.emitState('error', err);
      this.renderError(err);
    } finally {
      if (this.autoAbortController?.signal.aborted === false) {
        this.autoAbortController = null;
      }
    }
  }

  // 手动取消（比如用户点击了关闭按钮）
  cancel() {
    this.manualAbortController?.abort();
    this.autoAbortController?.abort();
    this.search.cancel(); // 使用 debounce 的 cancel 方法
    this.clearSuggestions();
    this.lastQuery = '';
    this.emitState('idle');
  }

  renderSuggestions(results) { /* ... */ }
  clearSuggestions() { /* ... */ }
  renderError(err) { /* ... */ }
  emitState(state, data) { /* 触发自定义事件或回调 */ }
}
```

## 总结与扩展

手写防抖、节流、call、apply、bind 是面试中最高频的"基础题"，但真正完整的实现要覆盖大量边界条件。理解它们不仅仅是死记硬背代码，而是深入理解 JavaScript 的 this 绑定机制、原型链、函数式编程和异步调度。

**值得进一步探索的方向**：

- **Lodash 源码的 debounce 完整形态**：阅读 `_.debounce` 的源码（约 350 行），理解 maxWait、leading/trailing 的全部组合以及如何通过定时器链来控制
- **ES7 装饰器**：如果项目使用 Class Component，用 ES7 Decorator 实现防抖/节流注解，让代码更声明式
- **requestAnimationFrame 节流**：对于动画相关的节流，用 rAF 替代 setTimeout 更为合适
- **Reflect.apply**：内部实现中可以用 `Reflect.apply(fn, context, args)` 替代 `fn.apply(context, args)`，后者在某些情况不支持

```javascript
// 使用 rAF 的节流（用于动画）
function throttleAnimation(fn) {
  let ticking = false;
  return function (...args) {
    if (!ticking) {
      ticking = true;
      requestAnimationFrame(() => {
        fn.apply(this, args);
        ticking = false;
      });
    }
  };
}
```

掌握这些手写题背后的原理，会让你在面试中的代码不仅"能运行"，而且"有思考"。
