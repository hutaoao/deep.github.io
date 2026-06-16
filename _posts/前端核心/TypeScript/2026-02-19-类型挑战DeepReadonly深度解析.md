---
title: "类型挑战：DeepReadonly深度解析"
date: 2026-02-19 10:00:00 +0800
categories: [前端核心, TypeScript]
tags:
  - TypeScript
  - 类型体操
  - 递归类型
description: "从递归类型编程角度深度解析DeepReadonly、DeepPartial、DeepRequired等类型挑战的实现，掌握TypeScript类型体操中递归思维的核心技巧。"
---

## 一句话概括

本文从零推导 TypeScript 中 `DeepReadonly`、`DeepPartial`、`DeepRequired` 等深度递归工具类型的实现原理，揭示递归类型、映射类型与条件类型在类型体操中的配合方式，帮你建立起类型层面的递归编程思维。

## 背景

TypeScript 内置的 `Readonly<T>` 工具类型广为人知——它把目标类型的所有属性标记为 `readonly`，阻止后续赋值修改。但 `Readonly<T>` 只处理第一层属性：

```typescript
type Config = {
  name: string;
  nested: { key: string; value: number };
};

type ReadonlyConfig = Readonly<Config>;
// { readonly name: string; readonly nested: { key: string; value: number } }
// nested 内部的 key/value 依然可写！

const cfg: ReadonlyConfig = { name: "a", nested: { key: "k", value: 1 } };
cfg.nested.key = "changed"; // ✅ 没有类型报错！Readonly 没有深入内部
```

这就是**浅只读**的局限。现实中的数据类型往往是嵌套的——API 响应体、配置对象、状态树——每一层都应该被保护。要解决这个问题，我们需要**递归类型**：类型自己调用自己，层层深入直到叶子节点。

## 概念与定义

在进入实现之前，先回顾三个核心工具：

### 映射类型（Mapped Type）

映射类型遍历对象的所有键，对每个值类型应用变换：

```typescript
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};
```

`keyof T` 获取所有键的联合，`T[K]` 是索引访问类型取值。

### 索引访问类型（Indexed Access Type）

```typescript
type NestedType = Config["nested"]; // { key: string; value: number }
```

### 条件类型与分发（Conditional Type & Distributive）

```typescript
type IsPlainObject<T> = T extends Record<string, any> ? true : false;
```

当泛型参数 `T` 是联合类型时，条件类型会自动分发（distribute），即对联合的每个成员独立求值后合并结果——这在递归中需要特别留意。

## 最小示例：DeepReadonly 的最简实现

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends Record<string, any>
    ? T[K] extends Function
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};
```

这个实现做了三件事：
1. 遍历 `T` 的所有键，加 `readonly`
2. 对值类型 `T[K]`，判断是否为对象（`Record<string, any>`）
3. 如果是对象且非函数，递归调用 `DeepReadonly<T[K]>`
4. 否则原样返回（基础类型、函数等）

``````

逐层展开示例：

```typescript
type Nested = {
  a: { b: { c: number } };
  d: string;
};

// DeepReadonly<Nested> 展开：
// {
//   readonly a: DeepReadonly<{ b: { c: number } }>
//   // → { readonly b: DeepReadonly<{ c: number }> }
//   //   → { readonly b: { readonly c: number } }
//   readonly d: string;
// }
// 最终：
// { readonly a: { readonly b: { readonly c: number } }; readonly d: string }
```

## 核心知识点拆解

### Readonly 单层实现的局限

内置 `Readonly<T>` 的核心代码只有一行 `{ readonly [P in keyof T]: T[P] }`。对于基本类型属性（`string`、`number`、`boolean`）足够了，但遇到对象值类型时，它只是把值类型的引用标记为 `readonly`，并不递归进入内部。用户仍然可以通过 `cfg.nested.key = "x"` 修改嵌套属性。

这也揭示了一个关键认知：**TypeScript 内置工具类型都是"浅层"的**——`Partial<T>`、`Required<T>`、`Readonly<T>` 无一例外。深度版本需要自己实现。

### DeepReadonly 实现（递归映射类型）

完善上面的最小实现，增加对数组和元组的处理：

```typescript
type DeepReadonly<T> = T extends Function
  ? T
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;
```

这里利用 `extends object` 同时覆盖普通对象和数组。但有一个副作用：`Date`、`Map`、`Set` 等内置对象会被拆开。更精确的处理是：

```typescript
type DeepReadonly<T> =
  T extends (infer U)[] ? readonly DeepReadonly<U>[] :
  T extends Function ? T :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;
```

### 处理数组类型（DeepReadonlyArray）

数组在 TypeScript 中也是对象，但它的键包含数值索引和 `length`、`push` 等方法。直接应用映射类型会把数组变成"数组样对象"而非真正的元组。正确的做法是先做条件分支判断：

```typescript
type DeepReadonly<T> =
  T extends any[] ?
    // 如果是数组/元组，递归处理每个元素
    { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T extends Function ? T
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;
```

测试：

```typescript
type Arr = [{ a: number }, string];
type DeepArr = DeepReadonly<Arr>;
// readonly [{ readonly a: number }, string]
// 元组的 length 属性也变为 readonly，这正是我们想要的
```

### 处理函数类型（只读参数）

函数类型不应该被"拆开"——遍历函数类型的属性没有意义。在上面的实现中，`extends Function` 分支直接返回 `T`，函数类型被原样保留。更深层的需求是让函数的参数也变成 `readonly`（即阻止修改参数）：

```typescript
// 只读参数类型
type DeepReadonlyFn<T extends (...args: any[]) => any> =
  (...args: DeepReadonly<Parameters<T>>) => ReturnType<T>;
```

但实践中，函数类型作为嵌套对象的属性值时，保持原样更合理。TypeScript 的 `Readonly` 内置类型也采用了同样的策略。

### DeepPartial 实现

与 `DeepReadonly` 对称——将 `readonly` 改为 `?`（可选）：

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepPartial<T[K]>
    : T[K];
};
```

逐层展开示例：

```typescript
type UserConfig = {
  db: { host: string; port: number };
  cache: { ttl: number };
};

type PartialConfig = DeepPartial<UserConfig>;
// {
//   db?: { host?: string; port?: number };
//   cache?: { ttl?: number };
// }

// 赋值任意子集：
const cfg1: PartialConfig = {};
const cfg2: PartialConfig = { db: { host: "localhost" } }; // ✅ port 可选
```

### DeepRequired 实现

和 `DeepPartial` 相反，去掉所有可选标记（`?`），同时递归深入：

```typescript
type DeepRequired<T> = {
  [K in keyof T]-?: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepRequired<T[K]>
    : T[K];
};
```

用 `-?` 语法去掉可选标记。注意这在对象中有递归引用或循环引用时报错——TypeScript 在递归深度上有上限（默认为 50 层左右）。

### 递归类型与类型实例化的性能考量

每个递归调用都触发一次类型实例化。深度嵌套的类型（如 JSON AST，10+ 层）会在编译期产生成千上万的类型节点。TypeScript 编译器会跟踪实例化深度，当超出限制时中断：

```typescript
// 深度超过约 50 层时触发 Type instantiation is excessively deep and possibly infinite
type DeepReadonly<T> = { readonly [K in keyof T]: DeepReadonly<T[K]> };
type DeepNested = DeepReadonly<{ a: { b: { c: ... 50 more ... } } }>;
```

优化策略：
1. **提前终止**：对基础类型和函数直接返回，避免无意义递归
2. **缓存**：TypeScript 编译器内置了类型缓存机制，相同类型参数不会重复实例化
3. **扁平化设计**：如果深度超过 10 层，考虑重构数据结构

## 实战案例：将嵌套 API 响应类型变为只读

真实业务场景：一个 GraphQL 或其他 API 返回的嵌套响应体，在 Redux/ 状态管理中应该保持只读，防止意外修改。

```typescript
// API 响应类型
interface ApiResponse<T> {
  status: number;
  message: string;
  data: T;
  meta?: {
    page: number;
    total: number;
    links: {
      prev: string | null;
      next: string | null;
    };
  };
}

// 用户数据类型
interface User {
  id: number;
  name: string;
  profile: {
    avatar: string;
    settings: {
      theme: "light" | "dark";
      notifications: boolean;
    };
  };
  posts: Array<{
    id: number;
    title: string;
    comments: Array<{
      id: number;
      text: string;
    }>;
  }>;
}

// 深度只读
type ReadonlyApiResponse = DeepReadonly<ApiResponse<User>>;
// 所有层级不可变——防止 reducer 误修改

// 使用示例：
function processResponse(res: ReadonlyApiResponse) {
  res.meta.page = 2;        // ❌ Error: readonly
  res.data.profile.settings.theme = "dark"; // ❌ Error: readonly
  res.data.posts[0].comments[0].text = "x"; // ❌ Error: readonly
  res.status = 200; // ❌ Error: readonly
}
```

如果只需要部分字段只读，可以结合 `Pick`：

```typescript
type ReadonlyDataOnly<T> = {
  readonly [K in keyof T]: K extends "data" | "meta" ? DeepReadonly<T[K]> : T[K];
};
```

在 Redux Toolkit 或 Zustand 中，这种深度只读类型可以当作 Store 的接口类型，从类型层面保证不可变更新的正确性。

## 底层原理

### TypeScript 类型实例化上限

TypeScript 编译器有一个硬编码的递归上限（`maximum call stack` 防溢出），通常在 **50 层** 左右。这个限制是针对 **类型实例化深度** 而非普通模板递归。当编写递归工具类型时，如果数据嵌套深度超过这个限制，编译会失败：

```typescript
// 报错：Type instantiation is excessively deep and possibly infinite
type ExDeep = DeepReadonly<{
  l1: { l2: { l3: { /* ... 50 层 ... */ } } };
}>;
```

你可以通过 `--noErrorTruncation` 或修改 `tsconfig.json` 的 `maxNodeModuleJsDepth` 等配置来缓解，但根本解决方法是优化类型设计——不要让真实数据深度超过一个合理阈值。

### 递归深度：条件类型 vs 映射类型

条件类型递归（`T extends U ? Deep<T> : never`）和映射类型递归（`{ [K in keyof T]: Deep<T[K]> }`）底层走不同的递归路径：

- **映射类型递归**：每个属性展开一个分支，属于"广度优先"的树形展开。宽度越大，编译器节点越多
- **条件类型递归**：典型尾部递归风格，深度优先。容易触发实例化深度限制

对于 JSON 类型的深度只读，映射类型递归更自然，且每个分支独立，不容易被深度限制卡住——只要单个路径不超过 50 层。

### 条件分发在递归中的行为

当泛型 `T` 是联合类型（如 `string | number | { a: 1 }`）时，条件类型会自动分发。在 DeepReadonly 中可能产生意外行为：

```typescript
type DeepReadonly<T> =
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } : T;

// 如果传入联合类型：
type Test = DeepReadonly<string | { a: number }>;
// 分发为：DeepReadonly<string> | DeepReadonly<{ a: number }>
// → string | { readonly a: number }
```

这通常符合预期，但在处理 `never` 时要小心——空联合的递归会得到 `never`。

## 高频面试题解析

### 面试题 1：实现 DeepMutable

与 `DeepReadonly` 相反，去掉所有 `readonly` 标记：

```typescript
type DeepMutable<T> = {
  -readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepMutable<T[K]>
    : T[K];
};

type Obj = { readonly a: { readonly b: number } };
type Mutable = DeepMutable<Obj>; // { a: { b: number } }
```

考察点：是否知道 `-readonly` 映射修饰符。

### 面试题 2：DeepNonNullable

递归移除所有层的 `null` 和 `undefined`：

```typescript
type DeepNonNullable<T> = {
  [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? NonNullable<T[K]>
      : DeepNonNullable<NonNullable<T[K]>>
    : NonNullable<T[K]>;
};
```

考察点：`NonNullable<T>` 的理解 + 递归设计中的空值处理策略。

### 面试题 3：DeepPick

递归地从嵌套对象中挑选指定键路径：

```typescript
// 简化的单路径版
type DeepPick<T, Path extends string> =
  Path extends `${infer K}.${infer Rest}`
    ? K extends keyof T
      ? { [P in K]: DeepPick<T[K], Rest> }
      : never
    : Path extends keyof T
      ? { [P in Path]: T[Path] }
      : never;

type Obj = { user: { profile: { name: string } } };
type Picked = DeepPick<Obj, "user.profile">;
// { user: { profile: { name: string } } }
```

考察点：模板字面量类型中的 `infer` 匹配 + 递归拆解字符串路径。

### 面试题 4：DeepReadonly 如何保证属性值不是原始类型时递归处理？

关键在 `extends object` 分支判断。面试官会问：如果 `T[K]` 是 `Date` 会怎样？正确的答案应该是——`Date extends object` 为 `true`，但 `Date` 不是普通的数据对象，我们不应该递归进入它的属性（比如 `getTime` 方法）。所以需要额外判断：

```typescript
type DeepReadonly<T> =
  T extends Date ? Readonly<T> :  // 给 Date 加一层 readonly 但不拆解
  T extends (infer U)[] ? readonly DeepReadonly<U>[] :
  T extends Function ? T :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;
```

或者更通用地判断是否是"原始对象"：

```typescript
// 只对 plain object 做深度递归
type IsPlainObject<T> =
  T extends Record<string, unknown> ? true : false;
```

### 面试题 5：DeepReadonly 处理 Symbol 和模板字面量键

`keyof T` 已经包含了 `string | number | symbol`，映射类型天然支持：

```typescript
const sym = Symbol("key");
type Obj = { [sym]: { nested: string }; "template-${id}": number };
type RO = DeepReadonly<Obj>;
// { readonly [sym]: { readonly nested: string }; readonly "template-${id}": number }
```

不需要额外处理，映射类型的 `[K in keyof T]` 已经覆盖所有键类型。

## 总结与扩展

本文从 `DeepReadonly` 入手，覆盖了递归类型、映射类型、条件类型在 TypeScript 类型体操中的核心用法。关键收获：

1. **递归思维**：每个递归类型都需要明确的"基础情况"（终止条件）和"递归情况"
2. **映射修饰符**：`readonly`、`?`、`-readonly`、`-?` 是控制属性标记的核心语法
3. **分支顺序**：先判断数组/函数/Date 等特殊情况，再处理普通对象
4. **深度限制**：合理控制嵌套深度，避免编译期溢出

### 推荐练习

type-challenges 仓库（github.com/type-challenges/type-challenges）提供了从简单到困难的一系列 TypeScript 类型挑战，推荐按以下顺序练习：

- **简单**：`Readonly`、`TupleToObject`、`Exclude`
- **中等**：`DeepReadonly`、`Chainable Options`、`Permutation`
- **困难**：`IsUnion`、`CamelCase`、`String to Number`

将每个挑战的"解题思路"写下来——为什么这样写，如果不这样写会怎样。这样做比刷 100 道题更有价值。

真正的类型体操能力，不是记住多少工具类型，而是面对一个陌生的类型需求时，能拆解出"我需要映射什么""什么时候递归""什么时候终止"。这是区分"会用 TypeScript"和"理解 TypeScript"的分水岭。

**最终脑图**：

```
DeepReadonly
├─ 基础：映射类型 + readonly
├─ 递归条件：T[K] extends object
├─ 终止条件：基础类型 / 函数 / Date
├─ 数组分支：readonly DeepReadonly<U>[]
└─ 扩展：
   ├─ DeepPartial：[K in keyof T]?
   ├─ DeepRequired：[K in keyof T]-?
   ├─ DeepMutable：-readonly
   └─ DeepNonNullable：递归 NonNullable
```
