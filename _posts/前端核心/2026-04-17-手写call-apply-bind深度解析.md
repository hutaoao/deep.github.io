---
title: "手写call apply bind深度解析"
date: 2026-04-17 08:00:00 +0800
categories: [前端核心, JavaScript]
tags: [JavaScript, call, apply, bind, this, 手写题, 面试题]
description: "从原理到实现，手写 call、apply、bind 三个方法，深入理解 this 显式绑定机制，掌握 bind 返回函数的 new 行为处理。"
---

## 一句话概括

`call`、`apply`、`bind` 都是用于**显式绑定 this** 的方法，核心思路是将函数作为目标对象的属性来调用，从而借助隐式绑定规则改变 this 指向。

---

## 背景

JavaScript 中 `this` 的指向是动态的，有时需要手动控制。`Function.prototype` 上的 `call`、`apply`、`bind` 三个方法提供了显式绑定能力。手写这三个方法是前端面试的经典题目，考察对 this 绑定规则、原型链、闭包的综合理解。

---

## 概念与定义

| 方法 | 调用方式 | 参数形式 | 是否立即执行 |
|------|---------|---------|------------|
| `call` | `fn.call(ctx, a, b)` | 逐个传参 | ✅ 立即执行 |
| `apply` | `fn.apply(ctx, [a, b])` | 数组传参 | ✅ 立即执行 |
| `bind` | `fn.bind(ctx, a, b)` | 逐个传参 | ❌ 返回新函数 |

---

## 最小示例

```javascript
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const user = { name: 'Alice' };

console.log(greet.call(user, 'Hello', '!'));    // "Hello, Alice!"
console.log(greet.apply(user, ['Hi', '~']));    // "Hi, Alice~"

const boundGreet = greet.bind(user, 'Hey');
console.log(boundGreet('?'));                    // "Hey, Alice?"
```

---

## 核心知识点拆解

### 1. call 的实现原理

核心思路：将函数临时挂载到目标对象上，利用隐式绑定调用，再删除该属性。

```javascript
Function.prototype.myCall = function(context, ...args) {
  // 1. context 为 null/undefined 时，指向全局对象
  context = context == null ? globalThis : Object(context);
  
  // 2. 用 Symbol 避免属性名冲突
  const fnKey = Symbol('fn');
  
  // 3. 将当前函数挂载到 context 上
  context[fnKey] = this;
  
  // 4. 调用函数（此时 this 指向 context）
  const result = context[fnKey](...args);
  
  // 5. 删除临时属性
  delete context[fnKey];
  
  return result;
};
```

### 2. apply 的实现原理

与 call 几乎相同，只是参数以数组形式传入：

```javascript
Function.prototype.myApply = function(context, args = []) {
  context = context == null ? globalThis : Object(context);
  const fnKey = Symbol('fn');
  context[fnKey] = this;
  const result = context[fnKey](...args);
  delete context[fnKey];
  return result;
};
```

### 3. bind 的实现原理

bind 返回一个新函数，且需要处理 **new 调用**的特殊情况：当 bind 返回的函数被 new 调用时，this 应指向新创建的对象，而不是绑定的 context。

```javascript
Function.prototype.myBind = function(context, ...bindArgs) {
  const originalFn = this;
  
  // 返回绑定函数
  function BoundFunction(...callArgs) {
    // 关键：如果被 new 调用，this 是 BoundFunction 的实例
    // 此时应忽略绑定的 context，使用 new 创建的对象
    const isNewCall = this instanceof BoundFunction;
    return originalFn.apply(
      isNewCall ? this : context,
      [...bindArgs, ...callArgs]
    );
  }
  
  // 维护原型链，使 instanceof 正常工作
  if (originalFn.prototype) {
    BoundFunction.prototype = Object.create(originalFn.prototype);
  }
  
  return BoundFunction;
};
```

---

## 实战案例

### 案例一：借用数组方法

```javascript
// 类数组对象借用数组方法
function sum() {
  // arguments 是类数组，没有 reduce 方法
  return Array.prototype.reduce.call(arguments, (acc, val) => acc + val, 0);
}
console.log(sum(1, 2, 3, 4)); // 10

// 将 NodeList 转为数组
const divs = document.querySelectorAll('div');
const divArray = Array.prototype.slice.call(divs);
```

### 案例二：bind 实现函数柯里化

```javascript
function multiply(a, b, c) {
  return a * b * c;
}

const double = multiply.bind(null, 2);    // 预填第一个参数
const triple = multiply.bind(null, 3);    // 预填第一个参数

console.log(double(3, 4));  // 24 (2 * 3 * 4)
console.log(triple(3, 4));  // 36 (3 * 3 * 4)
```

### 案例三：bind 返回函数被 new 调用

```javascript
function Point(x, y) {
  this.x = x;
  this.y = y;
}

const BoundPoint = Point.bind(null, 10); // 预填 x=10

const p = new BoundPoint(20); // new 调用，this 指向新对象
console.log(p.x); // 10
console.log(p.y); // 20
console.log(p instanceof Point); // true（原型链正确）
```

---

## 底层原理

### 为什么 bind 需要处理 new 调用？

根据 ECMAScript 规范，bind 返回的函数（BoundFunction）被 new 调用时：
1. 创建新对象，其原型指向**原始函数**的 prototype（不是 BoundFunction 的 prototype）
2. 执行原始函数，this 指向新对象
3. 绑定的 context 被**忽略**

这是为了保证 `new BoundFn()` 的行为与 `new OriginalFn()` 一致，只是预填了部分参数。

### Object(context) 的作用

当 context 是基本类型时，需要包装为对象：

```javascript
Function.prototype.myCall = function(context, ...args) {
  // 'hello'.myCall(42) → context = Number(42)
  context = context == null ? globalThis : Object(context);
  // ...
};
```

---

## 完整手写实现（面试版）

```javascript
// ===== myCall =====
Function.prototype.myCall = function(context, ...args) {
  context = context == null ? globalThis : Object(context);
  const key = Symbol();
  context[key] = this;
  const result = context[key](...args);
  delete context[key];
  return result;
};

// ===== myApply =====
Function.prototype.myApply = function(context, args = []) {
  context = context == null ? globalThis : Object(context);
  const key = Symbol();
  context[key] = this;
  const result = context[key](...args);
  delete context[key];
  return result;
};

// ===== myBind =====
Function.prototype.myBind = function(context, ...bindArgs) {
  const fn = this;
  function Bound(...args) {
    return fn.apply(
      this instanceof Bound ? this : context,
      [...bindArgs, ...args]
    );
  }
  if (fn.prototype) {
    Bound.prototype = Object.create(fn.prototype);
  }
  return Bound;
};

// ===== 测试 =====
function test(a, b) {
  console.log(this.name, a, b);
}
const obj = { name: 'test' };

test.myCall(obj, 1, 2);        // test 1 2
test.myApply(obj, [3, 4]);     // test 3 4
test.myBind(obj, 5)(6);        // test 5 6
```

---

## 高频面试题解析

### Q1：call 和 apply 的区别？

**答**：功能完全相同，唯一区别是参数传递方式：
- `call`：参数逐个传递 `fn.call(ctx, a, b, c)`
- `apply`：参数以数组传递 `fn.apply(ctx, [a, b, c])`

记忆技巧：**apply → array**，都以 a 开头。

### Q2：bind 与 call/apply 的区别？

**答**：
- `call`/`apply` 立即执行函数
- `bind` 返回一个新的绑定函数，不立即执行
- `bind` 支持参数预填充（偏函数应用）

### Q3：bind 返回的函数被 new 调用时，this 指向哪里？

**答**：指向 new 创建的新对象，绑定的 context 被忽略。这是规范规定的行为，目的是保证 new 的语义不被 bind 破坏。

### Q4：多次 bind 会怎样？

**答**：只有第一次 bind 的 context 生效，后续 bind 无法改变 this 指向：

```javascript
function foo() { console.log(this.x); }
const a = { x: 1 };
const b = { x: 2 };
const bound = foo.bind(a).bind(b);
bound(); // 1，第一次 bind 的 a 生效
```

### Q5：箭头函数可以使用 call/apply/bind 吗？

**答**：可以调用，但无法改变 this 指向。箭头函数的 this 在定义时就已确定（词法 this），call/apply/bind 传入的 context 会被忽略。

---

## 总结与扩展

```
call/apply/bind 核心思路：
  将函数挂载到目标对象 → 利用隐式绑定调用 → 删除临时属性

bind 的特殊处理：
  new 调用时 → 忽略绑定 context，使用新对象
  维护原型链 → BoundFn.prototype = Object.create(originalFn.prototype)
```

**扩展阅读**：
- `Reflect.apply(fn, ctx, args)` — ES6 提供的更规范的调用方式
- 函数柯里化与偏函数应用的区别
- `Function.prototype.toString()` — 查看函数源码
