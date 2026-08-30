---
layout: post
title: "WeakRef 与 FinalizationRegistry 弱引用实战"
date: 2026-11-08 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [WeakRef, FinalizationRegistry, 弱引用, 垃圾回收, 内存优化, 缓存]
description: >
  WeakRef 和 FinalizationRegistry 是 ES2021 直接触达 GC 的两个高级 API，面试常考"它和普通强引用、WeakMap 有什么区别""为什么不能用于关键清理"。
---

## 一句话概括

WeakRef 是对象的**弱引用**——它指向一个对象，但**不会阻止垃圾回收**。FinalizationRegistry 则让你在对象被回收后收到回调。说白了，这俩 API 是 JS 里唯一能"窥探垃圾回收"的口子。面试官问它，核心不是让你写缓存，而是考三点：**①弱引用和强引用的本质区别；②它和 WeakMap 的边界到底在哪；③为什么官方说"能不用就不用"，FinalizationRegistry 为什么不能用来做关键资源释放**。记住：它们是"内存优化工具"，不是"析构函数"。

## 核心知识点

### 1. 强引用 vs 弱引用：GC 计不计数

普通引用都是**强引用**：只要还有强引用指向对象，GC 就不会回收它。WeakRef 不计入这个计数——当对象只剩下弱引用时，GC 随时可以把它收走。

```js
// 强引用：obj 一直被 user 抓住，不会被回收
let user = { name: "John" };
let admin = user;      // 又一条强引用
user = null;           // 还有 admin 抓着，对象依然活着

// 弱引用：weak 不计入 GC 的引用计数
let target = { name: "Ada" };
const weak = new WeakRef(target);  // 只抓一条弱引用
target = null;                     // 再无强引用，对象可被回收
weak.deref()?.name;                // 活着就返回对象，被回收就返回 undefined
```

结论：**`weak.deref()` 的返回值你永远不能假设它是对象**——这一刻还在，下一刻可能就 `undefined` 了。对象像"薛定谔的猫"，你没法预先知道它死没死。

### 2. WeakRef 怎么用：只接受对象，每次取值都要判空

`new WeakRef(obj)` 只接受**对象**作目标，传原始值直接抛 `TypeError`。取值必须靠 `deref()`，并且永远要处理 `undefined` 分支：

```js
// ❌ 传原始值：直接 TypeError
new WeakRef(42);
new WeakRef("hello");

// ✅ 只接受对象，且每次用都要 guard
const ref = new WeakRef({ value: 42 });
const obj = ref.deref();                 // 可能是对象，也可能是 undefined
const v = obj ? obj.value : "已回收";     // 永远处理 undefined 分支
```

### 3. FinalizationRegistry：对象"死"后才会触发的回调

`register(obj, heldValue, unregisterToken?)` 注册一个对象，等它被回收后调用回调。注意几个**硬性坑**：

```js
const registry = new FinalizationRegistry((heldValue) => {
  console.log("被回收了：", heldValue);  // heldValue 是 register 时传的第二个参数
});

let big = { id: 1 };
registry.register(big, "big-1");  // 第二个参数 heldValue 会原样传给回调
big = null;                       // 之后某个不确定的 GC 周期，回调"才可能"执行
```

**坑一：回调时机完全不可控，甚至可能永远不执行。** 别拿它做必须发生的清理（关文件、释放锁、清定时器）。MDN 原话："Avoid using them where possible because the runtime semantics are almost completely unguaranteed."（能不用就不用，因为运行时语义几乎完全不受保证。）

**坑二：heldValue 如果传对象，registry 会对它保持强引用，反而挡住回收。** 这是最高频误区：

```js
// ❌ 把对象本身当 heldValue：registry 又抓了一条强引用，对象永远回收不了
registry.register(obj, obj);

// ✅ heldValue 用轻量标识（字符串/id），注册时顺便传第三个参数当注销令牌
registry.register(obj, obj.id, obj);  // 第三个参数 unregisterToken（弱引用）
// 想主动注销时：
registry.unregister(obj);            // 用同一个 token 即可
```

### 4. 它和 WeakMap 的边界：为什么缓存系统要用 Map + WeakRef

WeakMap 的**键**是弱引用，但**值**是强引用。这意味着：如果你想做"URL → 大对象"的缓存，**不能**用 WeakMap——因为你抓着 key（URL 字符串）就能拿到 value，value 就一直活着，缓存会无限膨胀。正确做法是普通 Map + WeakRef 值：

```js
// ❌ WeakMap 做缓存：值被强引用，大对象永不释放，缓存越积越大
const cache = new WeakMap();
function getExpensive(key) {            // key 还必须是对象，字符串当不了 WeakMap 键
  if (!cache.has(key)) cache.set(key, createExpensive(key));
  return cache.get(key);                // 对象永远被 value 抓住
}

// ✅ 普通 Map 装 WeakRef 值：对象别处不用了就能被 GC，下次 deref 拿到 undefined 重新算
function weakCache(getter) {
  const cache = new Map();
  const registry = new FinalizationRegistry((key) => {
    if (!cache.get(key)?.deref()) cache.delete(key);  // 回收后顺手清 Map 死条目
  });
  return (key) => {
    const ref = cache.get(key)?.deref();
    if (ref) return ref;                // 还活着，直接复用
    const value = getter(key);
    cache.set(key, new WeakRef(value)); // 值用弱引用包起来
    registry.register(value, key);      // 回收时清 Map 键
    return value;
  };
}
```

一句话区分：**WeakMap 关心"键别让 GC 留着"，WeakRef 关心"值能被 GC 收走"。** 想让 value 可释放，只能自己用 Map + WeakRef。

### 5. 何时该用 / 何时别碰

面试常让你判断场景，结论如下：

| 场景 | 用 WeakRef/FR？ | 理由 |
|---|---|---|
| 缓存大对象，希望没人用时自动释放 | ✅ 适合 | 内存优化，正是它设计目的 |
| 监听 DOM 节点是否还在文档里 | ✅ 适合 | 第三方库不挡节点回收 |
| 释放文件句柄 / 关数据库连接 | ❌ 别用 | 时机不可控，必须用 try/finally |
| 清定时器、解绑事件 | ❌ 别用 | 必须显式 cleanup |
| 普通数据共享 / 状态传递 | ❌ 别用 | 普通变量、强引用就够了 |

## 其实你每天都在用

- **大对象缓存**：图片/二进制 Blob 按 URL 缓存，希望没人用时自动释放，就是 WeakRef 的典型场景
- **框架内部追踪 DOM**：第三方监控/日志库用 WeakRef 跟踪你页面上的 DOM 节点，节点被移除也不会泄漏
- **Observer 模式里不让 listener 挡住 subject**：理论上可用 WeakRef 让监听器不阻止被观察者回收（多数库仍偏好显式 unsubscribe）
- **V8 / SpiderMonkey / JavaScriptCore** 内部都维护一张不参与可达性判断的弱引用表，WeakRef 就是暴露这张表的口子
- **排查内存泄漏时**：理解它，能帮你分清"是强引用没断，还是真的该用弱引用"

## 常见误解（FAQ）

**❌ 误区一："WeakRef 能阻止对象被回收，我拿它当保活引用"**

正好相反。WeakRef 的存在**不影响** GC 决策——只要没有强引用，对象该收就收，WeakRef 拦不住。它叫"弱"引用，使命就是"不保活"。

**❌ 误区二："FinalizationRegistry 回调像析构函数，对象一变不可达就立即执行"**

完全错误。回调**可能延迟几秒甚至更久**，也可能**永远不执行**（比如浏览器关标签页、进程退出时根本不跑）。所以它绝不适合做"必须发生"的清理。需要确定性清理，老老实实 `try { ... } finally { cleanup() }`。

**❌ 误区三："deref() 一定会返回对象，我直接 `.prop` 就行"**

`deref()` 在任意 GC 周期都可能返回 `undefined`。直接 `ref.deref().setting` 会 `TypeError`。正确姿势永远先判空：`const o = ref.deref(); o?.xxx`。

**❌ 误区四："WeakRef 和 WeakMap 是一回事，都是弱引用"**

差在"谁弱"。WeakMap 是**键弱**；WeakRef 是**引用本身弱**。想让"值"可释放（如缓存 value 是大对象），WeakMap 做不到，得 Map + WeakRef。这是面试题最爱挖的坑。

**❌ 误区五："register 时把对象本身当 heldValue 方便"**

register 的第二个参数 heldValue 如果是对象，registry 会对它保持**强引用**，等于又抓了对象一把，它永远回收不了。heldValue 该用轻标识（字符串/id），需要注销时再传第三个 token。

## 一句话总结

WeakRef 是"不保活"的弱引用、`deref()` 可能随时返回 `undefined`；FinalizationRegistry 是"对象死后不一定会响"的回调——它们只该做**非关键的内存优化**，永远别替代 `try/finally` 的确定性清理，且和 WeakMap 的边界是"键弱 vs 引用弱"。
