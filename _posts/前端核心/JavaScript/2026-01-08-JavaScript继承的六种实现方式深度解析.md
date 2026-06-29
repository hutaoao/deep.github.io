---
title: "JavaScript继承的六种实现方式深度解析"
date: 2026-01-08 08:00:00 +0800
categories: [前端核心, JavaScript]
tags: [JavaScript, 继承, 原型链, 面试题, 寄生组合继承]
description: "系统梳理 JavaScript 继承的六种实现方式，从原型链继承到寄生组合继承，揭示 ES6 class extends 的底层真相。"
---

## 一句话概括

JavaScript 继承本质上是对**原型链的精心编排**——六种方式构成一条清晰的演进路线，最终收敛于"寄生组合继承"，而这正是 `class extends` 的底层实现。

---

## 核心知识点

### 1. 原型链继承 — 把父类实例丢给子类 prototype

最直观的想法，也是最容易翻车的：

```js
function Parent() {
  this.colors = ['red', 'blue'];
}
Parent.prototype.sayHi = () => console.log('hi');

function Child() {}
Child.prototype = new Parent(); // 👈 核心

const c1 = new Child();
const c2 = new Child();
c1.colors.push('green');
console.log(c2.colors); // ['red', 'blue', 'green'] ← 共享了！
```

**致命伤**：引用类型被所有实例共享。一个改了，全部遭殃。

### 2. 构造函数继承 — call 一把梭

```js
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}

function Child(name, age) {
  Parent.call(this, name); // 👈 核心
  this.age = age;
}

const c1 = new Child('Tom', 5);
const c2 = new Child('Jerry', 3);
c1.colors.push('green');
console.log(c2.colors); // ['red', 'blue'] ✅ 不共享了
```

**致命伤**：原型上的方法拿不到——`c1.sayHi()` 报错。

### 3. 组合继承 — call + new，双保险

```js
function Child(name, age) {
  Parent.call(this, name);     // 第一次调 Parent
  this.age = age;
}
Child.prototype = new Parent(); // 第二次调 Parent 😱
Child.prototype.constructor = Child; // 顺手修 constructor
```

父类构造函数被调了**两次**。子类 prototype 上多了一套冗余的父类实例属性。

### 4. 原型式继承 — Object.create 的思想源头

```js
const parent = { colors: ['red'], sayHi() { console.log('hi'); } };
const child = Object.create(parent);
child.colors.push('blue');
// parent.colors 也变成 ['red', 'blue'] —— 共享问题依然在
```

适合"基于已有对象创建一个类似对象"的场景，不解决引用共享。

### 5. 寄生式继承 — 工厂函数包装一下

```js
function createChild(parent) {
  const obj = Object.create(parent);
  obj.walk = () => console.log('walking'); // 增强
  return obj;
}
```

只是给原型式继承包了一层工厂函数，额外的能力加上了，共享问题没解决。

### 6. 寄生组合继承 ⭐ — 终极方案

一把解决所有问题：

```js
function Parent(name) {
  this.name = name;
  this.colors = ['red', 'blue'];
}
Parent.prototype.sayHi = function () {
  console.log(`Hi, I'm ${this.name}`);
};

function Child(name, age) {
  Parent.call(this, name);   // 只调一次
  this.age = age;
}

// 核心两行
Child.prototype = Object.create(Parent.prototype);
Child.prototype.constructor = Child;

// 验证
const c1 = new Child('Tom', 5);
c1.sayHi();                          // Hi, I'm Tom
console.log(c1 instanceof Parent);   // true
```

这就是 Babel 把 `class extends` 编译后的样子。

---

## 「其实你每天都在用」

**1. React 类组件继承 `React.Component`**

```jsx
class MyComponent extends React.Component {
  render() { /* ... */ }
}
// Babel 编译后 → 寄生组合继承
```

**2. Array / Date 的继承困境**

```js
// 直觉写法 —— 但其实有坑
class MyArray extends Array {
  first() { return this[0]; }
}
// 背后：寄生组合 + Symbol.species 的复杂处理
```

**3. Vue 2 的 `Vue.extend()`**

源码里 `Vue.extend` 用原型链创建组件构造器——内部就是 `Sub.prototype = Object.create(Super.prototype)`。

**4. Node.js 的 `util.inherits`**

```js
const util = require('util');
function MyStream() { /* ... */ }
util.inherits(MyStream, require('stream').Stream);
// 底层也是 Object.create
```

**5. 任何 `new` 操作背后**

每次你写 `new Dog()`，JS 引擎在 `Dog.prototype` 上找方法——这一切都依赖继承建立的原型链。

---

## 常见误解（FAQ）

**❌ 误区 1：「ES6 class 和 Java 的 class 一样」**

Java 的 class 是静态模板，编译时确定；JS 的 class 只是语法糖，底层是运行时原型链。你在 `constructor` 里加个 `console.log(this.__proto__)` 就明白了。

**❌ 误区 2：「`Child.prototype = new Parent()` 和 `Object.create(Parent.prototype)` 效果一样」**

不一样。`new Parent()` 执行了构造函数，会在 `Child.prototype` 上留下一堆 `name`、`colors` 等实例属性，虽然是 `undefined`，但占据了属性位置。`Object.create` 干净得多——只建原型桥，不执行构造函数。

**❌ 误区 3：「`instanceof` 判断的是构造函数」**

`instanceof` 判断的是**原型链**上是否存在构造函数的 `prototype`。所以：
```js
function A() {}
function B() {}
A.prototype = Object.create(B.prototype);
new A() instanceof B; // true —— 即使 A 和 B 毫无关系
```

**❌ 误区 4：「寄生组合继承完美无缺」**

也有小瑕疵：需要在子类构造函数里手动 `Parent.call(this)`，还要记得修正 `constructor`。ES6 class 帮我们自动做了这两件事，所以如果你在用 ES6+，直接用 class。

---

## 一句话总结

**六种方式本质上是同一问题的六次迭代：从"能跑就行"到"跑得优雅"，每一步都在修补前人的坑，最终 ES6 用语法糖终结了这场接力赛。**

> 原型链共享有问题，构造函数借用解引用；组合继承两次调，寄生组合一次搞；`Object.create` 是关键，`constructor` 别忘改。
