---
layout: post
title: "手写 Partial / Required / Pick / Omit"
date: 2026-02-14
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "工具类型", "类型体操", "手写实现", "面试"]
---

## 一句话概括

**这四个工具类型加起来不到 10 行代码，却是 TypeScript 类型体操的"九九乘法表"**——面试写不出来基本告别高级岗，写出来后什么 `DeepPartial`、`Mutable`、`PartialByKeys` 都是它们的排列组合。

## 核心手写

### 1. Partial——全部变可选

```ts
type MyPartial<T> = {
  [P in keyof T]?: T[P];
};
// 关键：在属性名后加 ?
```

### 2. Required——全部变必填

```ts
type MyRequired<T> = {
  [P in keyof T]-?: T[P];
};
// 关键：-? 移除可选修饰符
// +? 是显式添加可选，? 省略 + 号也一样
```

### 3. Pick——选取指定属性

```ts
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};
// 注意：遍历的是 K 不是 keyof T
// 泛型约束 K extends keyof T 保证 K 的每个成员都是 T 的合法键
```

### 4. Omit——排除指定属性（两种写法都掌握）

```ts
// 方式一：as + never（推荐，直观）
type MyOmit<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};

// 方式二：Pick + Exclude（官方实现）
type MyOmit2<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// K extends keyof any（= string | number | symbol）比 K extends keyof T 更宽松
```

### 5. 组合拳——面试高频变体

```ts
// Readonly：全部变只读
type MyReadonly<T> = { readonly [P in keyof T]: T[P] };

// Mutable：移除只读
type Mutable<T> = { -readonly [P in keyof T]: T[P] };

// PartialByKeys：只把 K 中的属性变可选
type PartialByKeys<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
// 思路：未选中的保持不变，选中的 Partial 化，最后交叉合并

// RequiredByKeys：只把 K 中的属性变必填
type RequiredByKeys<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;
```

## 「其实你每天都在用」

1. **Partial<User>**：编辑表单，`onChange` 只更新改动的字段
2. **Pick<User, "id" | "name">**：列表接口只返回前端需要的字段
3. **Omit<Config, "password" | "secret">**：返回给客户端前脱敏
4. **Required<Props>**：确保组件调用方传了所有必需的 props
5. **`Partial<Pick<T, K>> & Omit<T, K>`** 的组合：最常见的是"编辑场景——某些字段可选某些必填"

## 常见误解（FAQ）

**❌ 误区 1：Omit 的 K 约束写 `keyof T` 就行**

写 `K extends keyof T` 也行，但官方用 `K extends keyof any` 更宽松，因为 `K` 可以是任意 string/number/symbol 的联合，即使里面有一些键在 T 中不存在也没关系——不存在的键排除不掉也无所谓。

**❌ 误区 2：`Partial<T> & Required<T>` 等于 T**

不对。交叉类型会合并属性，`{ name?: string } & { name: string }` 的结果是 `{ name: string }`（必填覆盖可选）。但这个"覆盖"不完美——某些边缘情况下可能会出现 `{ name?: string | undefined }`。

**❌ 误区 3：映射类型中 `[P in keyof T]` 和 `[P in K]` 是一回事**

不是。`[P in keyof T]` 遍历 T 的所有键（全量），`[P in K]` 只遍历 K 中的键（子集）。Pick 用后者，Partial/Required 用前者。

**❌ 误区 4：Omit 可以用 `[P in Exclude<keyof T, K>]` 直接实现**

TS 4.1 之前不能，因为映射类型不支持直接在 `in` 后面写条件。4.1 引入 `as` 重映射后才有了方式一。两种都掌握，面试讲出这个版本演进是加分项。

## 一句话总结

Partial 加 `?`，Required 减 `?`，Pick 遍历 K，Omit 排除 K——四者都是映射类型的变体，**掌握映射类型就掌握了所有内置工具类型的密码本**。
