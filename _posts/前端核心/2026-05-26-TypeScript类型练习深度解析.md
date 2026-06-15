---
title: TypeScript类型练习深度解析
date: 2026-05-26 09:00:00 +0800
categories: [前端核心, TypeScript]
tags: [TypeScript, 类型系统, 练习]
toc: true
---

> 一句话概括：本章通过接口定义、泛型使用、类型推断三条练习主线，系统巩固 TypeScript 类型系统的核心能力，建立类型化编程思维。

## 背景

TypeScript 的学习曲线中，类型系统是最大的一座山。很多人学完了语法，却在实际项目中不知道如何组织类型、不知道什么时候用 `interface` 什么时候用 `type`，也不知道如何让 TypeScript 帮你自动推断出精准的类型。本章通过真实场景练习，帮助你把语法知识转化为肌肉记忆。

## 概念与定义

### 接口 vs 类型别名

```typescript
// interface：声明对象结构的首选
interface User {
  id: number;
  name: string;
  email?: string;     // 可选属性
  readonly role: string; // 只读属性
}

// type：表达联合、交叉、原始类型
type ID = string | number;
type Pair<T> = [T, T];
type Callback = (err: Error | null, data: string) => void;
```

**选择原则：**
- 对象结构 → `interface`（未来可被 `implements`）
- 联合类型、条件类型 → `type`

## 核心知识点拆解

### 练习一：接口定义与继承

```typescript
// 基础接口
interface Animal {
  name: string;
  age: number;
}

// 接口继承（多重继承）
interface Dog extends Animal {
  breed: string;
  bark(): void;
}

// 用泛型实现可复用结构
interface APIResponse<T = unknown> {
  code: number;
  data: T;
  message: string;
}

// 练习：为以下 API 定义类型
// GET /users/:id -> { id, name, email, createdAt }
// GET /posts -> { items: Post[], total: number, page: number }
interface Post {
  id: number;
  title: string;
  content: string;
  authorId: number;
}

type UserResponse = APIResponse<{
  id: number;
  name: string;
  email: string;
  createdAt: string;
}>;

type PostListResponse = APIResponse<{
  items: Post[];
  total: number;
  page: number;
}>;
```

### 练习二：泛型约束

```typescript
// 泛型函数：约束必须有 length 属性
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length > b.length ? a : b;
}

longest('hello', 'world');        // ✅ string 有 length
longest([1, 2, 3], [1, 2]);      // ✅ array 有 length
longest({ length: 5 }, { length: 3 }); // ✅ 对象有 length
// longest(1, 2);                 // ❌ number 没有 length

// keyof 约束：确保 key 存在于对象中
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: 'Alice', age: 18 };
getProperty(user, 'name');   // ✅ 类型为 string
getProperty(user, 'age');   // ✅ 类型为 number
// getProperty(user, 'email');  // ❌ 'email' 不在 user 中
```

### 练习三：类型推断

```typescript
// 1. 基础推断
const data = { name: 'Bob', age: 30 };
// TypeScript 自动推断：{ name: string; age: number }

// 2. 函数返回类型推断
function process(x: number, y: number) {
  return { sum: x + y, product: x * y };
}
// 返回类型被推断为：{ sum: number; product: number }

// 3. 上下文推断
const handler = (event: MouseEvent) => {
  // event.target 被推断为 HTMLElement | null
  const target = event.target as HTMLElement;
  target.dataset.id;  // ✅  narrowing 后可安全访问
};

// 4. 推断数组元素类型
const mixed = [1, 'a', true];
// 推断为：(number | string | boolean)[]

// 5. infer 关键字：从条件类型中提取类型
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type F = () => Promise<string[]>;
type R = ReturnType<F>;  // string[]
```

## 实战案例

### 完整练习：构建类型安全的 HTTP 客户端

```typescript
// Step 1: 定义 HTTP 方法类型
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';

// Step 2: 定义请求配置泛型
interface RequestConfig<M extends HttpMethod, R = unknown> {
  url: string;
  method: M;
  params?: Record<string, string | number>;
  data?: M extends 'GET' ? never : unknown;
  headers?: Record<string, string>;
}

// Step 3: 定义响应类型
interface HttpResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

// Step 4: 封装请求函数（利用泛型实现类型安全）
async function request<M extends HttpMethod, R = unknown>(
  config: RequestConfig<M, R>
): Promise<HttpResponse<R>> {
  const { url, method, params, data, headers } = config;
  
  let fullUrl = url;
  if (params) {
    const qs = new URLSearchParams(
      Object.entries(params).map(([k, v]) => [k, String(v)])
    ).toString();
    fullUrl += `?${qs}`;
  }

  const res = await fetch(fullUrl, {
    method,
    headers: { 'Content-Type': 'application/json', ...headers },
    body: method !== 'GET' ? JSON.stringify(data) : undefined,
  });

  const result = await res.json();
  return { data: result as R, status: res.status, message: 'ok' };
}

// 使用示例
interface User { id: number; name: string; email: string; }
interface CreateUserDTO { name: string; email: string; }

// ✅ GET 请求自动推断返回类型
const users = await request({
  url: '/api/users',
  method: 'GET',
  params: { page: 1, size: 20 }
});
users.data  // 类型为 unknown，使用时需要断言或定义泛型参数

// ✅ 指定泛型参数，data 类型精确
const typedUsers = await request<User[], 'GET'>({
  url: '/api/users',
  method: 'GET',
});
typedUsers.data  // User[]

// ✅ POST 请求自动禁用 data 参数（TS 类型检查）
const newUser = await request<null, CreateUserDTO>({
  url: '/api/users',
  method: 'POST',
  data: { name: 'Alice', email: 'alice@example.com' }
});
```

## 底层原理

### 结构化类型系统

TypeScript 使用**结构化类型系统**（Structural Type System），而不是名义类型系统。这意味着只要两个类型的结构兼容，TypeScript 就认为它们是类型兼容的：

```typescript
interface Point2D { x: number; y: number; }
interface Vector2D { x: number; y: number; }

const p: Point2D = { x: 0, y: 0 };
const v: Vector2D = { x: 1, y: 1 };
p = v;  // ✅ 结构相同，可以赋值（鸭式辩型）
```

### 类型推断的工作原理

TypeScript 编译器在遍历 AST 时维护一个**类型环境**，每当遇到新的表达式，就尝试从表达式的值、使用上下文、类型注解三个方向综合推断其类型：

```typescript
const arr = [];    // arr: any[]（初始推断）
arr.push(1);       // arr: number[]（扩展推断）
arr.push('a');     // arr: (number | string)[]（联合扩展）
```

### 可赋值性检查

```typescript
type A = { x: number };
type B = { x: number; y: number };

// B 的结构更丰富，可以赋值给 A（A 的每个属性 B 都有）
const b: B = { x: 1, y: 2 };
const a: A = b;  // ✅

// 反过来不行（A 缺少 y）
const a2: A = { x: 1 };
// const b2: B = a2;  // ❌ Property 'y' is missing
```

## 高频面试题解析

### Q1：interface 和 type 的区别是什么？什么时候用哪个？

**答：** 两者在表达对象类型时几乎可以互换，但关键区别有三：① `interface` 可以被 `extends` 和 `implements` 继承；② `interface` 有声明合并特性（同名的多个 interface 会自动合并），`type` 不支持；③ `type` 可以表达联合类型、元组、原始类型、条件类型，`interface` 不行。**实操建议**：表达 API 结构、数据模型等对象类型时用 `interface`；表达复杂类型组合时用 `type`。

### Q2：TypeScript 如何实现函数重载的类型安全？

**答：** TypeScript 使用"函数签名列表 + 实现签名"模式：

```typescript
// 重载签名1
function greet(name: string): string;
// 重载签名2
function greet(name: string, age: number): string;
// 实现签名（必须兼容所有重载签名）
function greet(name: string, age?: number): string {
  if (age !== undefined) {
    return `Hello ${name}, you are ${age}`;
  }
  return `Hello ${name}`;
}

greet('Alice');           // ✅ 调用签名1
greet('Bob', 30);         // ✅ 调用签名2
// greet(123);             // ❌ 不匹配任何签名
```

### Q3：如何让 TypeScript 推断出更精确的数组元素类型？

**答：** 有三种策略：① **显式泛型**：`const arr: number[] = []`；② **字面量初始化**：`const arr = [1, 2, 3] as const` 将数组锁定为只读元组；③ **satisfies 操作符**（TS 4.9+）：`const config = { a: 1 } satisfies Record<string, number>` 既验证类型又保留字面量推断。

## 总结与扩展

### 核心练习要点

1. **接口是对象类型的首选**，类型别名用于更复杂的类型表达式
2. 泛型约束让通用函数在不丢失类型信息的前提下接受多种输入
3. `keyof` + `extends keyof` 是实现"类型安全的属性访问"的标准模式
4. TypeScript 的类型推断是渐进式的，可以从右向左、从左向右、从返回值推断
5. `as` 类型断言是最后的手段，尽量让 TypeScript 自己推断

### 实际工程建议

- 养成在函数参数上显式标注类型、返回值让 TS 推断的习惯
- 使用 `strict` 模式，避免 `any` 类型蔓延
- 对于复杂类型，创建 `types/` 目录集中管理全局类型定义
- 使用 `type-fest` 等工具库扩展内置工具类型能力
