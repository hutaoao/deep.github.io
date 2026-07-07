---
layout: post
title: "TypeScript 类型练习"
date: 2026-02-09
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型系统", "接口", "泛型", "类型推断", "面试"]
---

## 一句话概括

**TypeScript 的类型系统不是"看懂"了就会写的**——接口定义、泛型推导、keyof 组合这些必须上手敲出来，才能从"认识"变成"肌肉记忆"。

## 核心练习

### 练习 1：接口进阶——readonly、可选、联合

```ts
// 定义 User 接口：id 只读、email 可选、address 可以是对象或 null
interface User {
  readonly id: number;
  name: string;
  email?: string;
  roles: string[];
  address: { city: string; street: string } | null;
}

// 测试
const u: User = { id: 1, name: "Alice", roles: ["admin"], address: null };
// u.id = 2; // ❌ readonly
```

### 练习 2：接口继承链条

```ts
interface BaseEntity {
  readonly id: number;
  createdAt: Date;
  updatedAt: Date;
}
interface User extends BaseEntity {
  name: string;
  email: string;
}
interface AdminUser extends User {
  permissions: string[];
  level: number;
}
// AdminUser 拥有 id/createdAt/updatedAt/name/email/permissions/level 全部属性
```

### 练习 3：泛型栈

```ts
class Stack<T> {
  private items: T[] = [];
  push(item: T) { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  isEmpty(): boolean { return this.items.length === 0; }
}

const numStack = new Stack<number>();
numStack.push(1); numStack.push(2);
numStack.pop(); // 2
```

### 练习 4：泛型 + keyof —— 类型安全的 pick

```ts
function pick<T extends object, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach(k => { result[k] = obj[k]; });
  return result;
}
const p = { name: "Alice", age: 30, active: true };
const sub = pick(p, ["name", "age"]); // { name: string; age: number }
// pick(p, ["email"]); // ❌ 编译报错
```

### 练习 5：类型推断——你能看出下面每个变量的类型吗？

```ts
function getUser() { return { id: 1, name: "Alice", roles: ["admin"] as const }; }
const user = getUser();
// user: { id: number; name: string; roles: readonly ["admin"] } 👀

function first<T>(arr: T[]) { return arr[0]; }
const f = first([1, 2, 3]);
// f: number | undefined ← strictNullChecks 下越界返回 undefined
```

### 练习 6：条件类型——IsString

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<string>;          // true
type B = IsString<number>;          // false
type C = IsString<"hello">;         // true（字面量 extends string）
type D = IsString<string | number>; // boolean ← 分发条件类型！
```

### 练习 7：可辨识联合（Discriminated Union）

```ts
type Result =
  | { status: "success"; data: unknown }
  | { status: "error"; message: string; code: number };

function handle(r: Result) {
  if (r.status === "success") {
    console.log(r.data); // 这里 r 是成功分支 ✅
  } else {
    console.error(r.message, r.code); // 这里 r 是错误分支 ✅
  }
}
// switch 同理——switch(r.status) 每个 case 自动窄化
```

## 「其实你每天都在用」

1. **表单数据类型**：`FormConfig<T>` 约束表单字段名和值类型一一对应
2. **useState 泛型**：`useState<User | null>(null)` 就是泛型+联合类型
3. **axios 请求封装**：`api.post<UserDTO>("/user", data)` 返回值类型随 T 变
4. **Vue 的 defineProps**：`defineProps<{ title: string }>()` 本质就是接口定义
5. **路由参数的类型推导**：`useRoute<{ id: string }>()` 保证 params.id 不拼错

## 常见误解（FAQ）

**❌ 误区 1：interface 和 type 随便用，没区别**

interface 支持声明合并（多次定义自动合并），type 支持联合/交叉/映射。简单对象用 interface，复杂类型（联合、工具类型）用 type。面试常问。

**❌ 误区 2：`keyof any` 就是 `string`**

`keyof any` 等于 `string | number | symbol`。因为对象的键可以是这三种类型。`Omit<T, K extends keyof any>` 支持数值索引和 symbol 键，比 `K extends keyof T` 更通用。

**❌ 误区 3：`as const` 只是让数组 readonly**

`as const` 的作用是把宽类型收窄为字面量类型：`["a", "b"]` 的类型从 `string[]` 变成 `readonly ["a", "b"]`。这是实现"值推导类型"的关键。

**❌ 误区 4：类型推断总是对的**

TypeScript 有时会推断出比你期望更宽的类型。比如 `const arr = [1, "a"]` 推断为 `(string | number)[]` 而不是 `[number, string]`。需要精确时用 `as const` 或显式标注。

## 一句话总结

类型是写出来的，不是看出来的——关掉类型提示默写一遍 `Pick<T, K extends keyof T>`，比读十篇教程管用。
