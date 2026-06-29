---
title: "手写call apply bind深度解析"
date: 2026-01-10 08:00:00 +0800
categories: [前端核心, JavaScript]
tags: [JavaScript, call, apply, bind, this, 手写题, 面试题]
description: "从零手写 call、apply、bind，深入理解 this 显式绑定、Symbol 妙用、bind 的 new 行为处理。"
---

## 一句话概括

`call`、`apply`、`bind` 的核心思路完全一致：**把函数临时挂到目标对象上当方法调用，借"隐式绑定"的东风改变 this**。区别只在于传参方式和是否立即执行。

---

## 核心知识点

### 1. myCall — 理解"借调"本质

```js
Function.prototype.myCall = function (ctx, ...args) {
  ctx = ctx == null ? globalThis : Object(ctx); // null → globalThis
  const key = Symbol('fn');                     // 防冲突
  ctx[key] = this;                              // 挂上去
  const ret = ctx[key](...args);                // 借隐式绑定调用
  delete ctx[key];                              // 擦干净
  return ret;
};

const obj = { name: 'Alice' };
function greet(greeting) { return `${greeting}, ${this.name}`; }
console.log(greet.myCall(obj, 'Hi')); // Hi, Alice
```

三个关键点：`Symbol` 防属性冲突、`Object(ctx)` 处理基本类型、`ctx == null` 兼容 `null`/`undefined`。

### 2. myApply — 和 call 只有传参区别

```js
Function.prototype.myApply = function (ctx, args = []) {
  ctx = ctx == null ? globalThis : Object(ctx);
  const key = Symbol('fn');
  ctx[key] = this;
  const ret = ctx[key](...args);
  delete ctx[key];
  return ret;
};
```

### 3. myBind — 返回新函数 + 处理 new 调用

这是最考功底的。bind 返回的函数如果被 `new` 调用，绑定的 `this` 要**让位**给新创建的对象：

```js
Function.prototype.myBind = function (ctx, ...bindArgs) {
  const fn = this;

  function Bound(...callArgs) {
    // this instanceof Bound → 说明是通过 new 调用的
    return fn.apply(
      this instanceof Bound ? this : ctx,
      [...bindArgs, ...callArgs]
    );
  }

  // 维护原型链，new 出来的实例才能通过 instanceof 检查
  if (fn.prototype) {
    Bound.prototype = Object.create(fn.prototype);
  }
  return Bound;
};

// 验证 new 场景
function Point(x, y) { this.x = x; this.y = y; }
const BoundPoint = Point.myBind(null, 10);
const p = new BoundPoint(20);
console.log(p.x);                     // 10
console.log(p instanceof Point);     // true ✅
```

### 4. call / apply / bind 一张表

| 方法 | 立即执行 | 传参方式 | 返回值 |
|------|---------|---------|--------|
| `call` | ✅ | 逐个 `(ctx, a, b)` | 函数返回值 |
| `apply` | ✅ | 数组 `(ctx, [a, b])` | 函数返回值 |
| `bind` | ❌ | 逐个 `(ctx, a, b)` | 新函数 |

### 5. 箭头函数无法被改变 this

```js
const arrow = () => console.log(this.name);
arrow.call({ name: 'Alice' }); // this 仍然是外层作用域的 this
```

箭头函数的 `this` 在**定义时**固化，`call`/`apply`/`bind` 的第一个参数被忽略。

---

## 「其实你每天都在用」

**1. 把类数组转成真数组**

```js
function sum() {
  return Array.prototype.reduce.call(arguments, (a, b) => a + b, 0);
}
sum(1, 2, 3); // 6
```

**2. 把 NodeList 当数组用**

```js
const divs = document.querySelectorAll('div');
Array.prototype.slice.call(divs); // [...divs] 出现之前的经典写法
```

**3. React 组件中的 bind**

```jsx
class Button extends React.Component {
  constructor(props) {
    super(props);
    this.handleClick = this.handleClick.bind(this); // 经典模式
  }
}
```

**4. 函数柯里化**

```js
const double = (a, b) => a * b;
const double10 = double.bind(null, 10);
double10(5); // 50
```

**5. 借用 `Object.prototype.toString` 做类型判断**

```js
Object.prototype.toString.call([]);        // "[object Array]"
Object.prototype.toString.call(new Date()); // "[object Date]"
```

---

## 常见误解（FAQ）

**❌ 误区 1：「bind 后再 bind 能换 this」**

不能。只有第一次 `bind` 生效：
```js
const a = { x: 1 }, b = { x: 2 };
const fn = function () { console.log(this.x); };
fn.bind(a).bind(b)(); // 1，不是 2
```

**❌ 误区 2：「call / apply 对箭头函数有效」**

无效。箭头函数没有自己的 `this`，参数被静默忽略，不会报错。所以你永远不知道它"没生效"。

**❌ 误区 3：「bind 返回的函数和原函数一样」**

不一样。bind 返回的是一个新的"bound function"（规范中的 Bound Function Exotic Object），有自己的内部槽位。`fn.bind(ctx).bind(ctx2)` 是无效的，因为第二次 bind 操作的对象已经不是原始函数了。

**❌ 误区 4：「apply 比 call 慢」**

性能差异可忽略。选哪个看数据形态：参数已知用 `call`；参数在数组里用 `apply`。ES6 有了展开运算符 `fn(...args)` 后，大部分场景都可以替代 `apply`。

---

## 一句话总结

**借东风，借的就是"隐式绑定"这阵风——把函数挂到目标对象上，this 自然就对了；bind 多走一步，包一层 new 判断，保证 new 优先级不被打乱。**
