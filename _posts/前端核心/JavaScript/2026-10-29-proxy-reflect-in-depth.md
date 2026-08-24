---
layout: post
title: "Proxy 与 Reflect 深入应用"
date: 2026-10-29 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [Proxy, Reflect, 元编程, Vue3响应式, 数据校验, 弱引用]
description: >
  面试高频：Proxy 怎么拦截对象操作、为什么一定要配合 Reflect、receiver 是什么、Vue 3 为什么从 defineProperty 换成 Proxy，以及 Proxy 的边界与性能坑。
---

## 一句话概括

Proxy 是 ES6 的"元编程"原语：它像一个中间人，把对对象的读取、赋值、删除、函数调用、`new` 等操作全部拦截下来，让你有机会插手。Reflect 是和它成对的 API，每个 Proxy 的 trap 都能在 Reflect 里找到同名方法——在 trap 里调 Reflect 才是"正确地把默认行为转交回原对象"。记住一句话：**Proxy 负责拦截，Reflect 负责兜底**。

## 核心知识点

### 1. 基本形态：target + handler（traps）

```js
const target = { name: "Ada" };
const proxy = new Proxy(target, {
  get(target, prop, receiver) {
    console.log(`读取了 ${prop}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    console.log(`写入了 ${prop} = ${value}`);
    return Reflect.set(target, prop, value, receiver); // 必须返回 true
  },
});
proxy.name;     // 读取了 name → "Ada"
proxy.age = 18; // 写入了 age = 18
```

要点：

- `target` 是被代理的本体，`handler` 里的每个方法叫 **trap（陷阱）**。没定义的 trap 自动透传到 target。
- 一共有 **13 个 trap**：`get` / `set` / `has` / `deleteProperty` / `ownKeys` / `getOwnPropertyDescriptor` / `defineProperty` / `getPrototypeOf` / `setPrototypeOf` / `isExtensible` / `preventExtensions` / `apply`（函数调用）/ `construct`（`new`）。
- `set` 必须 `return true`（严格模式下不返回会抛 `TypeError`），推荐直接 `return Reflect.set(...)`。

### 2. 为什么 trap 里要用 Reflect，而不是 `target[prop]`

直接 `target[prop] = value` 看似也行，但有两个坑：

1. **不返回规范的布尔值**：`Reflect.set` 失败（比如冻结对象）返回 `false`，手写的 `target[prop]=value` 静默成功，语义不一致。
2. **this 绑定错误（最容易翻车）**：当对象上有 getter 或涉及继承时，`this` 应该是 proxy 而不是 target，否则继承链上的 getter 会拿到错误的 `this`。

```js
// ❌ 直接用 target：继承 getter 里的 this 指向错
const parent = { get value() { return this._v * 2; } };
const child = Object.create(parent);
child._v = 5;
const bad = new Proxy(child, { get(t, p) { return t[p]; } });
console.log(bad.value); // 可能拿到 NaN / undefined

// ✅ 用 Reflect 并传 receiver，保证 getter 里的 this 是 proxy
const good = new Proxy(child, {
  get(t, p, receiver) { return Reflect.get(t, p, receiver); },
});
console.log(good.value); // 10，正确地以 proxy 为 this
```

一句话：**在 trap 里一律用 `Reflect.xxx(target, ..., receiver)`，把 receiver 原样透传**，这是框架级代码的铁律。

### 3. Proxy.revocable：用完即焚的临时授权

```js
const { proxy, revoke } = Proxy.revocable({ secret: 42 }, {});
proxy.secret; // 42
revoke();      // 吊销
proxy.secret;  // TypeError: Cannot perform 'get' on a revoked proxy
```

非常适合"临时把对象交给别人、用完立刻收回权限"的场景（一次性令牌、沙箱里的临时对象、限时访问）。

### 4. Vue 3 为什么用 Proxy 取代 Object.defineProperty

这是面试必问。Object.defineProperty 的两个硬伤：

- **监听不到新增/删除属性**：`data` 上后来加的字段 `obj.newKey = 1` 不会被劫持，Vue 2 要靠 `Vue.set` 兜底。
- **数组和深层对象要递归遍历**：初始化时把所有属性逐个 `defineProperty`，是 O(n)，对象一大就慢。

Proxy 一次性代理整个对象，读取时收集依赖、写入时触发更新，**天然支持新增/删除属性、原生支持数组索引修改**，初始化是 O(1)，不用一开始就深度遍历：

```js
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key, receiver) {
      track(target, key);                   // 收集依赖
      return Reflect.get(target, key, receiver);
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver);
      trigger(target, key);                 // 触发更新
      return result;
    },
  });
}
```

## 其实你每天都在用

- **Vue 3 响应式**：`ref()` / `reactive()` 返回的其实就是 Proxy，你改 `state.count`，视图自动更新，底层就是 get 收集依赖、set 触发更新。
- **数据校验**：表单对象包一层 Proxy，在 `set` 里拦截类型/范围，非法值直接抛错，业务代码完全无感。
- **默认值和虚拟字段**：`get` 时 `prop in target` 为假就返回默认值，或拼出 `fullName` 这种"算出来的属性"。
- **日志/调试**：给任意对象包一层 Proxy，所有读写自动打日志，不用动原对象一行代码。
- **防篡改/只读**：`set` / `deleteProperty` 直接返回 false 或抛错，把普通对象变成只读。

## 常见误解（FAQ）

**❌ 误区一："Proxy 可以完全替代原对象，谁也看不出区别"**

错。Proxy 是**另一个对象**，身份不同：`proxy === target` 是 `false`，`===` 这个操作 Proxy 拦不住。任何靠引用相等判断的地方（比如 Set/Map 去重、框架内部 identity 比较）都会把 proxy 和 target 当成两个东西。

**❌ 误区二："原生对象（Map/Date/DOM）也能随便套 Proxy"**

错。Map、Set、Date、Promise、DOM 元素这些有**内部插槽（internal slots）**，Proxy 无法透明转发。比如 `new Proxy(new Map(), {})` 后访问 `proxy.size` 会直接抛 `TypeError`。要代理它们得用"this 重定向"的特殊写法，否则别碰。

**❌ 误区三："Proxy 性能无所谓，业务里随便包一层"**

Proxy 每次操作都要走 trap，属性访问比直接访问慢几倍。热路径、大循环、高频调用的对象别包 Proxy——React 的 memo 缓存里误用 Proxy 是常见的性能事故来源。

**❌ 误区四："Reflect 就是个语法糖，和 target[prop] 没区别"**

Reflect 不是语法糖：它返回**规范的布尔成功状态**（set 失败返回 false 而不是静默），并且正确传递 receiver 处理继承 getter，还提供 `Reflect.apply` / `Reflect.construct` 等统一入口。在 trap 里用 Reflect 才能 100% 还原默认语义。

## 一句话总结

Proxy 是"拦截对象操作的中间人"，Reflect 是"正确兜底默认行为的工具"——**拦截用 trap，兜底用 Reflect，记得把 receiver 透传**；它撑起了 Vue 3 的响应式，但别拿去代理有内部插槽的原生对象，也别塞进性能热点。
