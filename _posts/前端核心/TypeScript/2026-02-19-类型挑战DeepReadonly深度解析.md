---
layout: post
title: "类型挑战：DeepReadonly"
date: 2026-02-19 00:00:00 +0800
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型体操", "DeepReadonly", "递归类型", "面试题"]
---

## 一句话概括

`DeepReadonly<T>` 把对象及其所有嵌套子对象的属性全部递归标记为 `readonly`。它是 type-challenges 的中等难度标杆题，考的是递归类型 + 映射类型 + 边界条件处理的组合能力。

## 核心知识点

### 1. 最简实现——先跑通再说

```ts
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>
    : T[K];
};
```

看起来不错，但埋了一个大坑 ↓

### 2. 函数类型——必须单独处理

```ts
type IsObject<T> = T extends object ? true : false;
type Test = IsObject<() => void>; // true！函数也是 object！

// ❌ 错误示范
type Fn = () => void;
// DeepReadonly<Fn> 会把 call、bind、apply 全部变成 readonly，彻底变形
```

正确的做法是**在进入递归前拦截函数类型**：

```ts
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K]                           // 函数直接保留
    : T[K] extends object
      ? DeepReadonly<T[K]>
      : T[K];
};
```

### 3. 数组和元组的处理

```ts
// 方案：数组判断放在最前面
type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>    // 递归元素
  : T extends (...args: any[]) => any
    ? T
    : T extends object
      ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
      : T;

// 验证
type Test = DeepReadonly<{ items: string[] }>;
// { readonly items: readonly string[] }
```

**注意顺序**：`T extends any[]` 必须写在 `T extends object` 前面，否则数组会被对象分支吞噬。

### 4. DeepPartial / DeepRequired —— 对称四件套

```ts
type DeepPartial<T> = T extends (infer U)[]
  ? DeepPartial<U>[]
  : T extends Function ? T
  : T extends object
    ? { [K in keyof T]?: DeepPartial<T[K]> }
    : T;

type DeepRequired<T> = T extends (infer U)[]
  ? DeepRequired<U>[]
  : T extends Function ? T
  : T extends object
    ? { [K in keyof T]-?: DeepRequired<T[K]> }  // 注意 -?
    : T;
```

理解 DeepReadonly 之后，DeepPartial / DeepRequired 只是改一个修饰符而已。

### 5. 编译期 + 运行时双保险——DeepFreeze

```ts
function deepFreeze<T>(obj: T): DeepReadonly<T> {
  Object.freeze(obj);
  for (const key of Object.keys(obj)) {
    const val = (obj as any)[key];
    if (val && typeof val === 'object' && !Object.isFrozen(val)) {
      deepFreeze(val);
    }
  }
  return obj as any;
}

const config = deepFreeze({ server: { host: 'localhost', port: 3000 } });
// config.server.port = 8080;  ❌ 编译报错 + 运⾏静默失败
```

## 其实你每天都在用

- **Redux / Zustand 的 reducer state**：state 必须是只读的，`.user.name = 'new'` 能直接被 TS 拦住
- **Vue 3 的 `defineProps`**：传入的 props 内部被 `DeepReadonly` 包裹，子组件改 props 直接类型报错
- **前端配置对象**：`appConfig.api.baseUrl` 运行时不该被改——DeepReadonly 在编译期就拦住
- **React.memo 的 props 比较**：只读类型让开发者不敢直接 mutate，只能用不可变更新，配合 memo 更可靠
- **环境变量对象**：`import.meta.env` 是只读的，TS 类型把它标的明明白白

## 常见误解

- **❌ 误区：「Readonly 就够用了」** `Readonly` 只浅层修饰。五层嵌套的对象，只有第一层属性被保护，剩下 80% 的属性照改不误。只要有一层嵌套，就必须上 DeepReadonly。

- **❌ 误区：「函数也是 object，为什么要跳过？」** 跳过函数不是偷懒，是必须。一旦对函数类型调用 `DeepReadonly`，TS 会遍历函数的属性（call、apply、bind……），产生一堆无意义的 readonly 属性，而且深层嵌套的函数会导致 TS 报递归深度超限。

- **❌ 误区：「数组不用单独处理」** 如果用映射类型遍历数组，`keyof []` 会包含所有数组方法和索引，结果类型会变成 `{ readonly 0: ...; readonly length: ...; readonly push: ... }`——这不是你想要的数组类型。应该把数组转为 `ReadonlyArray` 再递归元素。

- **❌ 误区：「DeepReadonly 会拖慢类型检查」** 对于 3-5 层的普通配置对象几乎没有性能影响。但如果是递归深度 40+ 层的自引用类型（如 AST 节点），TS 会报 "excessively deep"。此时应使用带深度限制的版本 `DeepReadonly<T, Depth extends number = 5>`。

## 一句话总结

DeepReadonly 的正解 = 数组在最前 → 函数拦截 → 对象递归映射 → 原始类型兜底，四个分支顺序一个不能错。
