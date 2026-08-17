---
layout: post
title: "手写 Partial / Required / Pick / Omit"
date: 2026-02-14
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "工具类型", "类型体操", "手写实现", "面试"]
---

## 一句话概括

**这四个工具类型加起来不到 10 行代码，却是 TypeScript 类型体操的"九九乘法表"**——面试手写不出来基本告别高级岗，写出来后 `DeepPartial`、`PartialByKeys`、`Mutable` 都是它们的排列组合。

## 核心手写

### 1. Partial — 全部属性变可选

```ts
type MyPartial<T> = {
  [P in keyof T]?: T[P];
};

// 使用
interface User { id: number; name: string; email: string; }
type Draft = MyPartial<User>;
// { id?: number; name?: string; email?: string }
```

**关键**：`[P in keyof T]?` 中的 `?` 给遍历出的每个属性加上可选修饰符。

### 2. Required — 全部属性变必填

```ts
type MyRequired<T> = {
  [P in keyof T]-?: T[P];
};

// -? 移除可选修饰符，? 前面加 - 表示"去掉"
// +? 是显式加可选（等价于直接写 ?）
```

### 3. Pick — 从 T 中选取指定属性

```ts
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

interface User { id: number; name: string; email: string; }
type Preview = MyPick<User, "id" | "name">;
// { id: number; name: string }
```

**两个关键点**：
- `K extends keyof T` — 约束 K 必须是 T 的合法键名，拼错编译报错
- 遍历的是 `K` 而非 `keyof T` — 只保留 K 涉及的属性

### 4. Omit — 从 T 中排除指定属性

```ts
// 方法一：as + 条件类型（TS 4.1+，直观）
type MyOmit<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};
// as never → TS 自动移除值为 never 的属性键

// 方法二：Pick + Exclude 组合（官方实现）
type MyOmitAlt<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
// 先 Exclude 排除要删的键，再 Pick 剩下的

// K extends keyof any (= string | number | symbol)
// 比 K extends keyof T 更宽松：可以传入 T 中不存在的键
```

### 5. 面试高频变体——组合拳

```ts
// Readonly：只读
type MyReadonly<T> = { readonly [P in keyof T]: T[P] };

// Mutable：移除只读
type Mutable<T> = { -readonly [P in keyof T]: T[P] };

// DeepPartial：递归把所有层级变可选
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object
    ? T[P] extends Function ? T[P] : DeepPartial<T[P]>
    : T[P];
};

// PartialByKeys：只让指定的属性变可选
type PartialByKeys<T, K extends keyof T> =
  Omit<T, K> & Partial<Pick<T, K>>;
// 思路：未选中的原样保留（Omit），选中的 Partial 化（Pick + Partial），交叉合并

// RequiredByKeys：只让指定的属性变必填
type RequiredByKeys<T, K extends keyof T> =
  Omit<T, K> & Required<Pick<T, K>>;
```

## 其实你每天都在用

1. **`Partial<User>`** — 编辑表单，只传用户改了的字段，其余字段后端用原值
2. **`Pick<User, "id" | "name">`** — 列表页只拿展示需要的字段，避免网络传输浪费
3. **`Omit<ApiConfig, "secret" | "password">`** — 台返回时自动脱敏，类型层面保证安全
4. **`Required<ComponentProps>`** — 底层组件所有 prop 必传，不给默认值
5. **`Record<"id" | "name", string>`** — 要 key-value 字典但不想手写 interface

## 常见误解（FAQ）

**❌ 误区 1：「Omit 的 K 写 `keyof T` 就行」**

官方的 `Omit<T, K>` 用 `K extends keyof any`（= `string | number | symbol`），而不是 `keyof T`。原因：K 可以包含 T 中不存在的键——不存在的键排除不掉无所谓，这是宽松设计，不是 bug。而 `K extends keyof T` 如果传入不存在的键会直接报错。

**❌ 误区 2：「`Partial<T> & Required<T>` 等于 T」**

交叉类型中必填覆盖可选，`{ name?: string } & { name: string }` → `{ name: string }`。但这个"覆盖"不完美——在部分 TS 版本中边缘情况下可能出现 `{ name: string | undefined }`。不要依赖交叉类型做"抵消"，应该直接重建映射类型。

**❌ 误区 3：「Pick 和 Omit 是相反操作，`Pick<T, keyof T> = Omit<T, never>`」**

对，但不完全等价。Pick 的泛型约束 `K extends keyof T` 保证 K 的每个成员都是合法键名。Omit 的约束 `K extends keyof any` 允许瞎传键名——这是故意的设计差异。

**❌ 误区 4：「映射类型的属性顺序和原对象一样」**

TS 的映射类型不保证属性顺序。`{ [P in keyof T]: T[P] }` 遍历 `keyof T`（联合类型），而联合类型的遍历顺序是不确定的。依赖映射类型的属性顺序是不安全的。

## 一句话总结

Partial 加 `?`，Required 减 `?`，Pick 遍历 K，Omit 排除 K——四个工具类型都是同一个公式 `[P in Something]: T[P]` 的参数化变体。**掌握映射类型就掌握了所有内置工具类型的源代码。**
