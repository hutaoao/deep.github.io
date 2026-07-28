---
layout: post
title: "TypeScript 类型练习"
date: 2026-02-09
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型系统", "接口", "泛型", "类型推断", "面试"]
---

## 一句话概括

**类型不是看会的，是敲会的**——接口、泛型、keyof、条件类型这些必须上手实现一遍，把手感练出来，面试时才能不卡壳。

## 核心练习

### 练习 1：接口进阶——Readonly、Optional、深层嵌套

```ts
// 需求：用户接口，id 不可改，email 可选，address 可空但结构固定
interface User {
  readonly id: number;
  name: string;
  email?: string;
  roles: string[];
  address: { city: string; street: string } | null; // 要么有地址对象，要么 null
}

const u: User = {
  id: 1,
  name: "Alice",
  roles: ["admin"],
  address: null, // ✅
};
// u.id = 2; // ❌ readonly
```

**面试追问**：`readonly` 和 `const` 的区别？—— readonly 管属性（编译时），const 管变量绑定（运行时）。

### 练习 2：接口继承链——三层能力层层叠加

```ts
interface BaseEntity {
  readonly id: number;
  createdAt: Date;
}

interface User extends BaseEntity {
  name: string;
  email: string;
}

interface AdminUser extends User {
  permissions: string[];
  level: number;
}

// AdminUser 现在有: id, createdAt, name, email, permissions, level
const admin: AdminUser = {
  id: 1, createdAt: new Date(),
  name: "Alice", email: "a@b.com",
  permissions: ["write"], level: 3,
};
```

### 练习 3：泛型栈——类型安全的 push/pop

```ts
class Stack<T> {
  private items: T[] = [];

  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); } // 空栈返回 undefined
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  get size(): number { return this.items.length; }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
console.log(numStack.pop());  // 2
console.log(numStack.peek()); // 1
```

### 练习 4：泛型 + keyof——手写类型安全的 Object.pick

```ts
function pick<T extends object, K extends keyof T>(
  obj: T,
  keys: K[]
): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach(k => {
    result[k] = obj[k];
  });
  return result;
}

const user = { name: "Alice", age: 30, active: true };
const sub = pick(user, ["name", "age"]); // { name: string; age: number }
// pick(user, ["email"]); // ❌ 编译报错——email 不存在

// 面试常问：为什么约束是 K extends keyof T 而不是 K extends string？
// 答：保证 K 的每个成员都是 T 的真实键名，而不仅仅是任意字符串
```

### 练习 5：类型推断——猜猜每个变量是什么类型

```ts
// Q1：getUser 没有显式返回类型，推断出来是什么？
function getUser() {
  return { id: 1, name: "Alice", roles: ["admin"] as const };
}
const user = getUser();
// user: { id: number; name: string; roles: readonly ["admin"] }
// 注意：as const 让 roles 从 string[] 收窄为 readonly ["admin"]！

// Q2：first 函数的返回值类型？
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
const f = first([1, 2, 3]);
// f: number | undefined —— strictNullChecks 下越界返回 undefined

// Q3：空数组传泛型
const e = first([]);
// e: undefined，且 T 推断为 never
```

### 练习 6：条件类型——手写 IsString、Exclude、Extract

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<"hello">;  // true
type B = IsString<42>;       // false
type C = IsString<string | number>; // boolean ← 分发条件类型！

// Exclude：从 T 中排除 U
type MyExclude<T, U> = T extends U ? never : T;
type D = MyExclude<"a" | "b" | "c", "a">; // "b" | "c"

// Extract：从 T 中提取 U
type MyExtract<T, U> = T extends U ? T : never;
type E = MyExtract<"a" | "b" | "c", "a" | "c">; // "a" | "c"
```

### 练习 7：可辨识联合（Discriminated Union）

```ts
type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function render<T>(state: AsyncState<T>): string {
  switch (state.status) {
    case "idle":    return "待加载";
    case "loading": return "加载中...";
    case "success": return `数据: ${JSON.stringify(state.data)}`; // state.data 可用 ✅
    case "error":   return `异常: ${state.error.message}`;        // state.error 可用 ✅
    default: {
      const _exhaustive: never = state; // 穷尽检查：新增分支漏了这里会报错
      return _exhaustive;
    }
  }
}
```

## 「其实你每天都在用」

1. **API 响应处理**：`if (res.code === 0) { res.data.items.map(...) }` — `res.data` 的类型根据 code 窄化
2. **React `useState`**：`useState<User | null>(null)` — 紧跟状态的泛型保证 getter/setter 类型一致
3. **Vue `defineProps`**：`defineProps<{ title: string; count?: number }>()` — 纯类型定义替代运行时 validator
4. **axios 封装**：`function post<T>(url: string, data: unknown): Promise<ApiResponse<T>>` — 一层泛型省掉所有调用处的 as 断言
5. **路由类型推导**：`useRoute<{ id: string }>()` — 保证 `params.id` 不拼错、类型不猜

## 常见误解（FAQ）

**❌ 误区 1：「interface 和 type 就是写法不同，功能一样」**

interface 支持声明合并（`interface User { name: string }; interface User { age: number }` 自动合并为 `{ name: string; age: number }`），type 不支持。type 支持联合类型、交叉类型、映射类型，interface 不支持。简单对象用 interface，复杂类型组合用 type——两者互补而非替代。

**❌ 误区 2：「`as const` 只是让数组变成 readonly」**

`as const` 的核心作用是**把宽类型收窄为字面量类型**。`{ role: "admin" }` 类型是 `{ role: string }`，而 `{ role: "admin" } as const` 类型是 `{ readonly role: "admin" }`。这是"从值推导精确类型"的关键，做类型体操离不开它。

**❌ 误区 3：「类型推断永远是对的，不用显式标注」**

TS 有时推断得比你期望更宽。`const arr = [1, "a"]` 推断为 `(string | number)[]` 而非 `[number, string]`；`Object.keys()` 返回 `string[]` 而非 `(keyof T)[]`。关键边界的返回类型应该显式标注，不依赖推断。

**❌ 误区 4：「`keyof any` 就是 `string`」**

`keyof any` = `string | number | symbol`。`Omit<T, K>` 的 K 约束为 `keyof any` 而非 `keyof T`，是因为你可以传入 T 中不存在的键（反正不存在就不用删）。这也是为什么用 `K extends keyof any` 比 `K extends keyof T` 更宽松。

## 一句话总结

关掉 AI 补全，从空白文件默写一遍 `Pick<T, K extends keyof T>` 和 `Exclude<T, U>` — 写到肌肉记忆才叫真的会了。
