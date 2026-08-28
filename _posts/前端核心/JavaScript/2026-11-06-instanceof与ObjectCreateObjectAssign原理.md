---
layout: post
title: "instanceof 与 Object.create / Object.assign 原理（手写）"
date: 2026-11-06 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [instanceof, Object.create, Object.assign, 原型链, 手写实现]
description: >
  面试高频手写题：instanceof、Object.create、Object.assign 的底层原理与手写实现，重点讲清原型链查找、null 原型陷阱与 Get/Set 拷贝语义。
---

## 一句话概括

这三个 API 是面试手写区的常客，本质都绕不开**原型链**和**属性描述符**两条主线：`instanceof` 问的是"构造函数的 `prototype` 在不在对象的原型链上"；`Object.create` 问的是"怎么以指定对象为原型造一个新对象"；`Object.assign` 问的是"怎么把源对象的可枚举自有属性浅拷贝到目标上"。说白了，它们都不是黑魔法，读懂规范里的几条规则，你就能在白板上还原 80% 的正确实现，剩下的 20% 是面试官爱追问的边界坑（原始值、`null` 原型、Symbol 键、getter 求值）。

## 核心知识点

### 1. instanceof：沿着原型链找构造函数的 prototype

面试官怎么问："手写一个 `myInstanceof`，实现原生 `instanceof` 的行为。"

你怎么答：核心就一句话——**不断取对象的 `__proto__`（即 `Object.getPrototypeOf`），看哪一步等于 `right.prototype`；走到 `null` 还没找到就返回 `false`**。

```js
// ✅ 标准手写的骨架：沿原型链向上爬
function myInstanceof(left, right) {
  // 原始值（number/string/boolean/null/undefined）直接返回 false（原生 instanceof 不装箱）
  if (left === null || (typeof left !== 'object' && typeof left !== 'function')) return false;
  if (typeof right !== 'function') {
    throw new TypeError('Right-hand side of instanceof is not callable');
  }
  let proto = Object.getPrototypeOf(left); // 从对象的原型开始
  const prototype = right.prototype;       // 目标构造函数的 prototype
  while (proto) {
    if (proto === prototype) return true; // 找到了
    proto = Object.getPrototypeOf(proto); // 没找到就往上一层
  }
  return false;
}

// 验证
function Foo() {}
const f = new Foo();
myInstanceof(f, Foo);    // true
myInstanceof(f, Object); // true（Foo.prototype 的原型是 Object.prototype）
myInstanceof({}, Array); // false
```

两个必须知道的边界：

```js
// ❌ 坑一：字符串字面量不是 String 的实例
'abc' instanceof String;        // false（字面量是原始值，原型链上根本没有 String.prototype）
new String('abc') instanceof String; // true

// ✅ 坑二：instanceof 可被 Symbol.hasInstance 自定义
class Fake {
  static [Symbol.hasInstance](obj) {
    return obj && obj.tag === 'fake'; // 不看原型链，只看标记
  }
}
({ tag: 'fake' }) instanceof Fake; // true
```

记住：**`instanceof` 的结果不是一成不变的**——`Foo.prototype = {}` 之后再 `new Foo()`，老对象的 `instanceof Foo` 可能就变成 `false` 了。

### 2. Object.create：以 proto 为新对象的原型

面试官怎么问："实现 `myCreate(proto)`，让返回对象的 `__proto__` 指向传入的 `proto`。"

你怎么答：经典思路是用一个空函数当跳板——把它的 `prototype` 设成目标对象，再 `new` 一下，得到的实例原型自然就是它。

```js
// ✅ 经典实现
function myCreate(proto) {
  function F() {}
  F.prototype = proto; // 关键：让 new F() 出来的实例以 proto 为原型
  return new F();
}

const parent = { say() { console.log('hi'); } };
const child = myCreate(parent);
child.say();                 // 'hi'（通过原型链访问到）
Object.getPrototypeOf(child) === parent; // true
```

但面试官若追问"`Object.create(null)` 行不行"，上面这个写法就露馅了：

```js
// ❌ 经典写法的隐藏 bug：proto 为 null 时，new F() 的原型会退化成 Object.prototype
function F() {}
F.prototype = null;
const obj = new F();
Object.getPrototypeOf(obj) === null; // false！退化成了 Object.prototype

// ✅ 正确处理 null 原型：用 setPrototypeOf
function myCreate(proto) {
  if (proto !== null && typeof proto !== 'object' && typeof proto !== 'function') {
    throw new TypeError('Object prototype may only be an Object or null');
  }
  const obj = {};
  Object.setPrototypeOf(obj, proto); // 支持 null 原型（Object.create(null) 的场景）
  return obj;
}
Object.getPrototypeOf(myCreate(null)) === null; // true ✅
```

原生 `Object.create` 还有第二个参数（属性描述符），面试答到 `Object.defineProperties(obj, propertiesObject)` 就是加分项：

```js
function myCreate(proto, propertiesObject) {
  const obj = {};
  Object.setPrototypeOf(obj, proto);
  if (propertiesObject != null) {
    Object.defineProperties(obj, propertiesObject); // 按描述符定义自身属性
  }
  return obj;
}
const o = myCreate({ x: 1 }, { y: { value: 2, enumerable: true } });
o.y; // 2
```

### 3. Object.assign：浅拷贝可枚举自有属性

面试官怎么问："手写 `myAssign(target, ...sources)`，对齐 `Object.assign`。"

你怎么答：把规则拆开就三句话——**只拷自有可枚举属性（字符串键 + Symbol 键）；逐源覆盖（后面的 source 赢）；修改并返回 target；`null/undefined` 的 source 直接跳过，但 `null/undefined` 的 target 要抛错**。最关键的一点：拷贝是 `Get + Set` 语义，不是复制描述符——源的 getter 会被求值，目标的 setter 会被触发。

```js
// ✅ 对齐原生行为的核心实现
function myAssign(target, ...sources) {
  if (target == null) { // ⚠️ 只有 target 是 null/undefined 才抛错
    throw new TypeError('Cannot convert undefined or null to object');
  }
  const to = Object(target); // 原始值 target 会被装箱（如字符串变成 String 包装对象）
  for (const source of sources) {
    if (source == null) continue; // ⚠️ null/undefined 的源直接跳过
    // Reflect.ownKeys = 字符串键 + Symbol 键，且都是"自有"
    for (const key of Reflect.ownKeys(source)) {
      if (Object.prototype.propertyIsEnumerable.call(source, key)) {
        to[key] = source[key]; // Get(source) + Set(to)：getter 求值、setter 触发
      }
    }
  }
  return to; // 返回被改过的 target（同一引用）
}

// 验证关键行为
const t = { a: 1 };
myAssign(t, { b: 2 }, { a: 9, c: 3 }); // 返回 t，且 t === 返回值
t; // { a: 9, b: 2, c: 3 } —— 后面的源覆盖前面的

// ❌ 最容易讲错的细节：它拷贝的是"值"不是"描述符"
const src = { get val() { return Date.now(); } }; // 源是 getter
const copy = myAssign({}, src);
typeof copy.val; // 'number' —— getter 已被求值，目标拿到的是普通数据属性
```

## 其实你每天都在用

- **`instanceof`**：`[] instanceof Array`、`new Date() instanceof Date` 判断类型；React 里 `error instanceof Error` 区分异常类型
- **`Object.create(null)`**：做纯字典/Map 时用，避免原型链上的 `toString`、`hasOwnProperty` 等键污染（前端配置表常这么干）
- **`Object.create(proto)`**：寄生组合继承的核心——`Child.prototype = Object.create(Parent.prototype)` 建立原型链又不调父类构造函数
- **`Object.assign`**：`Object.assign({}, default, userOpts)` 合并默认配置；Redux 旧写法里 `Object.assign({}, state, { count: 1 })` 做不可变更新
- **`Object.assign` 合并默认参数**：组件 props 缺省值兜底，几乎是每个工具的标配

## 常见误解（FAQ）

**❌ 误区一："instanceof 为 true，就说明对象一定是 new 这个构造函数造出来的"**

不一定。只要对象的原型链上有 `Foo.prototype`，哪怕你手动 `obj.__proto__ = Foo.prototype`，`instanceof Foo` 也是 `true`。真正的"品牌校验"要用私有字段或 `Symbol.hasInstance`。

**❌ 误区二："Object.create 就是浅拷贝一个对象"**

不是。`Object.create(proto)` 创建的是**空对象**，只是把 `proto` 设为它的原型；它本身没有任何自有属性，你访问到的属性全在原型上。要拷贝属性得配合 `Object.assign`。

**❌ 误区三："Object.assign 是深拷贝"**

浅拷贝。嵌套对象只是引用复制，改了源里的嵌套对象，目标里也会跟着变。需要深拷贝请用 `structuredClone` 或手写递归。

**❌ 误区四："Object.assign 会连不可枚举属性和 Symbol 键一起拷"**

只拷**可枚举的自有属性**，包含 Symbol 键，但**不包含**不可枚举属性、继承属性，也**不复制属性描述符**（getter 会被求值）。要连描述符一起拷，得用 `Object.defineProperties(target, Object.getOwnPropertyDescriptors(source))`。

**❌ 误区五："手写 instanceof 时左边是原始值会抛错"**

不会抛，直接返回 `false`。只有**右边不是函数**（且没有 `Symbol.hasInstance`）时才抛 `TypeError`。这是很多人手写的实现里漏掉的第一步判断。

## 一句话总结

`instanceof` 是"沿原型链找 constructor.prototype"，`Object.create` 是"以指定对象为原型造空对象（注意 `null` 原型要用 setPrototypeOf）"，`Object.assign` 是"把源的可枚举自有属性按 Get/Set 浅拷到目标"——三个 API 的本质都写在原型链和属性描述符这两条规则里。
