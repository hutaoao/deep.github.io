---
layout: post
title: "Set/Map/WeakSet/WeakMap 与内存管理"
date: 2026-10-30 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [Set, Map, WeakMap, WeakSet, 弱引用, 内存泄漏, 垃圾回收]
description: >
  面试高频：Set/Map 怎么用、WeakMap/WeakSet 的"弱引用"到底是什么、为什么它们不能遍历也没有 size、什么时候该用弱引用避免内存泄漏。
---

## 一句话概括

Set 是"去重集合"，Map 是"键可以是任意类型的对象"。WeakMap/WeakSet 是它们的"弱引用版"：**键/值只持有弱引用，对象一旦没别的地方引用就会被垃圾回收，条目自动消失**。记住一句话：**普通版防不住 GC（可能内存泄漏），Weak 版不挡 GC（自动清理）**——后者专门用来给对象挂元数据、做缓存、存私有数据，而不拖累对象的生命周期。

## 核心知识点

### 1. Set：唯一值集合

```js
const nums = [1, 2, 2, 3, 3, 3];
const unique = [...new Set(nums)]; // [1, 2, 3]

// 集合运算（ES2024 起原生支持）
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);
a.union(b);        // Set {1, 2, 3, 4}
a.intersection(b); // Set {2, 3}
a.difference(b);   // Set {1}
```

要点：

- 成员唯一、可迭代、有 `.size`、可 `.clear()`。
- 判断"是否存在"用 `set.has(x)`，**比数组 `includes` 快几千倍**（O(1) vs O(n)）。
- 面试坑：Set 用 `Object.is` 判等，`NaN` 和 `NaN` 算相等，`+0` 和 `-0` 算相等；对象是按引用去重，两个长得一样的对象不是同一个。

### 2. Map：键任意类型的键值对

```js
const m = new Map();
m.set("name", "Ada");     // 字符串键
m.set(1, "数字键");        // 数字键
m.set(() => {}, "函数键");  // 函数也能当键
m.set({ id: 1 }, "对象键"); // 对象也能当键，且各自独立
m.size;                    // 4
```

为什么不用普通对象做映射？普通对象的键**一律被转成字符串**，`obj[{ a: 1 }]` 实际键是 `"[object Object]"`，多个对象会互相覆盖；Map 的键保持原类型，还能用对象/函数当键，且**保持插入顺序**、自带 `.size`。

### 3. WeakMap：键是对象 + 弱引用

```js
const cache = new WeakMap();
function heavy(obj) {
  if (cache.has(obj)) return cache.get(obj);
  const result = obj.value * 2; // 假设昂贵计算
  cache.set(obj, result);
  return result;
}
let data = { value: 10 };
heavy(data); // 计算
data = null; // 没别的地方引用 data 了 → GC 回收它，cache 里那条也跟着消失
```

关键区别（对比 Map）：

| 维度 | Map | WeakMap |
|---|---|---|
| 键类型 | 任意（含原始值） | **只能是对象** |
| 引用强度 | 强引用，键挡 GC | 弱引用，键不挡 GC |
| 可迭代 | 是（keys/values/forEach） | **否** |
| `.size` | 有 | **无** |
| `.clear()` | 有 | **无** |
| 方法 | get/set/has/delete + 遍历 | 只有 get/set/has/delete |

**为什么键必须是对象？** 原始值（数字/字符串）是按值比较的，引擎无法判断"所有对 42 的引用都没了"；对象按引用比较，引擎能精确知道它何时不可达。所以 WeakMap 只接受对象键——这是弱引用能成立的物理前提。

**为什么不能遍历、没有 size？** 因为 GC 随时可能回收条目，暴露 `.size` 或迭代会得到不确定的结果（不同引擎、甚至同一引擎不同 GC 周期结果都不同），规范干脆砍掉，换来"自动清理"的确定性语义。

### 4. WeakSet：只能放对象的"标记册"

```js
const visited = new WeakSet();
function traverse(obj) {
  if (typeof obj !== "object" || obj === null || visited.has(obj)) return; // 防循环引用
  visited.add(obj);
  for (const k of Object.keys(obj)) traverse(obj[k]);
}
```

WeakSet 只存对象、弱引用、不能遍历也没有 size，常用场景：

- **标记"已访问/已处理"**：图遍历防死循环（上面例子）、避免重复初始化。
- **品牌检查（branding）**：给某构造器产出的对象打标，`brand.has(obj)` 判断"这对象是不是我造的"。

### 5. 典型误用：用普通 Map 缓存 DOM 导致内存泄漏

```js
// ❌ 内存泄漏：节点从 DOM 移除后，Map 还强引用它，永远清不掉
const meta = new Map();
function track(el) { meta.set(el, { clicks: 0 }); }

// ✅ 用 WeakMap：节点被移除且无其他引用 → 自动回收，连带 metadata 一起没
const meta2 = new WeakMap();
function track2(el) { meta2.set(el, { clicks: 0 }); }
```

React / Vue 内部就是用 WeakMap 把组件状态挂靠到 fiber / VNode 上，组件卸载时不会留下一堆孤儿数据。

## 其实你每天都在用

- **数组去重**：`[...new Set(arr)]` 是最常用的去重一行流。
- **去重 + 存在判断**：接口返回列表做 `Set` 判重、`Set.has` 做权限/标签校验，比数组快得多。
- **Vue 3 响应式**：用 WeakMap 把"响应式代理"和"原始对象"互相关联，组件销毁时自动解绑，不泄漏。
- **DOM 元数据**：给按钮挂点击次数、给元素挂滚动位置，用 WeakMap 存，元素删了数据自动没。
- **私有数据**：类实例的密码等敏感字段放 WeakMap（以 `this` 为键），外部拿不到，实例被回收时一起清。

## 常见误解（FAQ）

**❌ 误区一："WeakMap 没有 size，我用 .size 拿个长度不行吗"**

不行，WeakMap 压根没有这个属性，`wm.size` 是 `undefined`（不报错，但拿不到）。要计数得自己额外维护一个普通 Map 或计数器。

**❌ 误区二："用字符串当 WeakMap 的键"**

直接 `TypeError: Invalid value used as weak map key`。WeakMap 的键必须是对象，原始值（含字符串、数字、symbol）都不行。需要原始值当键就老老实实用普通 Map。

**❌ 误区三："WeakMap 能遍历，我 for...of 一下"**

不能，没有迭代器。弱引用集合的设计目标就是"不可枚举"，这是规范刻意砍掉的。想遍历就别用 Weak 系列。

**❌ 误区四："WeakMap 是来替代 Map 的，更省内存所以都用它"**

错。WeakMap / WeakSet 牺牲了 size、遍历、clear，换来了"自动 GC"。只有当你**明确希望对象的生命周期独立于数据、且不需要枚举**时才用它（DOM 元数据、私有数据、对象缓存）。需要遍历、计数、原始值键、长久保存——一律用普通 Map / Set。

## 一句话总结

Set 去重、Map 任意键映射；WeakMap / WeakSet 是弱引用版，**键/值不挡垃圾回收、自动清理、不可遍历也没有 size**——专门给"对象挂元数据/缓存/私有数据且不想拖累其生命周期"的场景用，用错地方（想要 size、要遍历）就退回普通 Map / Set。
