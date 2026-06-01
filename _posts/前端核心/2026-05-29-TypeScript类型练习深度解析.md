---
title: "TypeScript类型练习深度解析"
date: 2026-05-29
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型系统", "接口", "泛型", "类型推断"]
description: "通过接口定义、泛型使用、类型推断的实战练习巩固TypeScript核心类型知识"
---

## 一句话概括

TypeScript 的类型系统不是"学"会的，是"练"会的——通过有针对性的练习，将接口定义、泛型使用、类型推断从知识转化为肌肉记忆。

## 背景

前四天我们已经学习了 TypeScript 的基础类型系统、接口与类型别名、泛型基础、枚举等核心概念。但"理解"和"会用"之间有一道鸿沟：

- 看懂别人写的类型 ≠ 自己能写出正确的类型
- 记住语法 ≠ 知道什么时候用什么
- 单个概念理解 ≠ 多个概念组合使用

这篇文档是**实战练习**，覆盖面试中最常见的类型编程场景，每个练习都包含：
- 题目描述
- 初始代码骨架
- 参考答案
- 关键知识点标注

## 概念与定义

### 练习的三条原则

1. **先自己写**：看答案前至少尝试 5 分钟
2. **理解为什么**：不是记住答案，而是理解每一行的作用
3. **举一反三**：修改练习条件，验证理解是否到位

### 练习分类

| 类别 | 核心能力 | 面试出现频率 |
|------|---------|-------------|
| 接口定义 | 描述数据结构、类型组合 | ⭐⭐⭐⭐⭐ |
| 泛型使用 | 类型参数化、约束与推断 | ⭐⭐⭐⭐⭐ |
| 类型推断 | 理解 TypeScript 推断规则 | ⭐⭐⭐⭐ |
| 综合练习 | 多概念组合运用 | ⭐⭐⭐⭐⭐ |

## 最小示例

```typescript
// 练习0：热身——给函数添加类型
function add(a, b) {
  return a + b;
}
// 答案：
function add(a: number, b: number): number {
  return a + b;
}

// 练习0.5：热身——给对象添加类型
const user = {
  name: "Alice",
  age: 30,
};
// 答案：
interface User {
  name: string;
  age: number;
}
const user: User = {
  name: "Alice",
  age: 30,
};
```

## 核心知识点拆解

### 第一部分：接口定义练习

#### 练习1：用户信息接口

```typescript
// 定义一个 User 接口，满足以下需求：
// 1. id 是只读的数字
// 2. name 是字符串
// 3. email 是可选的字符串
// 4. roles 是字符串数组
// 5. address 可以是对象或 null

// 你的代码：

// 参考答案：
interface User {
  readonly id: number;
  name: string;
  email?: string;
  roles: string[];
  address: {
    city: string;
    street: string;
    zipCode: string;
  } | null;
}

// 测试
const user1: User = {
  id: 1,
  name: "Alice",
  roles: ["admin"],
  address: { city: "Beijing", street: "Chaoyang Rd", zipCode: "100000" },
};

const user2: User = {
  id: 2,
  name: "Bob",
  roles: ["user"],
  address: null,
};

// user1.id = 999;  // ❌ readonly 不允许修改
```

**关键知识点**：`readonly`、可选属性 `?`、联合类型、嵌套接口

#### 练习2：接口继承与合并

```typescript
// 定义基础接口和扩展接口：
// 1. BaseEntity: id(readonly number), createdAt(Date), updatedAt(Date)
// 2. User 继承 BaseEntity: name(string), email(string)
// 3. AdminUser 继承 User: permissions(string[]), level(number)

// 你的代码：

// 参考答案：
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

// 测试
const admin: AdminUser = {
  id: 1,
  createdAt: new Date(),
  updatedAt: new Date(),
  name: "Alice",
  email: "alice@example.com",
  permissions: ["read", "write", "delete"],
  level: 3,
};
```

**关键知识点**：接口继承 `extends`、属性继承链

#### 练习3：索引签名与 Record

```typescript
// 定义一个配置对象类型，满足：
// 1. 任意字符串键
// 2. 值必须是 string | number | boolean
// 3. 必须有 version 字段（string）

// 你的代码：

// 参考答案：
interface Config {
  version: string;
  [key: string]: string | number | boolean;
}

// 或者用交叉类型
type Config2 = {
  version: string;
} & Record<string, string | number | boolean>;

// 测试
const config: Config = {
  version: "1.0.0",
  debug: true,
  maxRetries: 3,
  logLevel: "info",
};
```

**关键知识点**：索引签名、Record 工具类型、交叉类型

### 第二部分：泛型使用练习

#### 练习4：泛型栈

```typescript
// 实现一个类型安全的栈类：
// 1. push(item: T): void
// 2. pop(): T | undefined
// 3. peek(): T | undefined
// 4. size(): number
// 5. isEmpty(): boolean

// 你的代码：

// 参考答案：
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  size(): number {
    return this.items.length;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}

// 测试
const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
numberStack.push(3);
console.log(numberStack.pop());   // 3
console.log(numberStack.peek());  // 2
console.log(numberStack.size());  // 2

const stringStack = new Stack<string>();
stringStack.push("hello");
stringStack.push("world");
console.log(stringStack.pop());   // "world"
```

**关键知识点**：泛型类、私有属性、数组方法

#### 练习5：泛型函数组合

```typescript
// 实现 compose 函数：将两个函数组合成一个
// compose(f, g)(x) 应该等价于 f(g(x))
// 要求类型安全

// 你的代码：

// 参考答案：
function compose<A, B, C>(
  f: (arg: B) => A,
  g: (arg: C) => B
): (arg: C) => A {
  return (x) => f(g(x));
}

// 测试
const double = (n: number): number => n * 2;
const toString = (n: number): string => n.toString();
const len = (s: string): number => s.length;

// double ← toString ← number
const doubleStringOfNumber = compose(double, len);
// 等价于: (x: number) => double(len(toString(x)))... 不对，类型不匹配

// 正确的组合链
const stringLen = compose(len, toString);  // (n: number) => number
console.log(stringLen(42));  // 2 (因为 "42" 长度为 2)

const doubleLen = compose(double, stringLen);  // (n: number) => number
console.log(doubleLen(42));  // 4
```

**关键知识点**：多泛型参数、函数类型表达、泛型推断链

#### 练习6：带约束的泛型——键值对提取

```typescript
// 实现一个函数 pick，从对象中选取指定的键：
// pick(obj, keys) 返回只包含指定键的新对象
// 要求：keys 中的键必须是 obj 的键

// 你的代码：

// 参考答案：
function pick<T extends object, K extends keyof T>(
  obj: T,
  keys: K[]
): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach(key => {
    result[key] = obj[key];
  });
  return result;
}

// 测试
const person = { name: "Alice", age: 30, active: true };

const nameAndAge = pick(person, ["name", "age"]);
// 类型: Pick<{ name: string; age: number; active: boolean }, "name" | "age">
// 即: { name: string; age: number }

// pick(person, ["email"]);  // ❌ "email" 不是 person 的键
```

**关键知识点**：`keyof`、`extends keyof`、`Pick` 工具类型

### 第三部分：类型推断练习

#### 练习7：推断返回值类型

```typescript
// 给出以下函数，写出每个变量的类型

function getUser() {
  return { id: 1, name: "Alice", roles: ["admin"] as const };
}

function getFirstItem<T>(arr: T[]) {
  return arr[0];
}

function parseJSON(str: string) {
  return JSON.parse(str) as { code: number; data: unknown };
}

// 推断以下变量的类型：
// 1. const user = getUser();
// 2. const first = getFirstItem([1, 2, 3]);
// 3. const firstStr = getFirstItem(["a", "b"]);
// 4. const result = parseJSON('{"code":200,"data":null}');

// 参考答案：
// 1. user: { id: number; name: string; roles: readonly ["admin"] }
// 2. first: number | undefined
// 3. firstStr: string | undefined
// 4. result: { code: number; data: unknown }

// 注意：as const 让 roles 成为字面量元组类型
// arr[0] 可能越界，所以返回 T | undefined（strictNullChecks）
```

**关键知识点**：`ReturnType`、`as const`、`strictNullChecks`、类型断言

#### 练习8：条件类型推断

```typescript
// 实现一个 IsString 类型：
// 如果 T 是 string，返回 true；否则返回 false

// 你的代码：

// 参考答案：
type IsString<T> = T extends string ? true : false;

// 测试
type A = IsString<string>;       // true
type B = IsString<number>;       // false
type C = IsString<"hello">;      // true（"hello" extends string）
type D = IsString<string | number>;  // boolean（分发条件类型）

// 进阶：阻止分发
type IsStringNoDistribute<T> = [T] extends [string] ? true : false;
type E = IsStringNoDistribute<string | number>;  // false
```

**关键知识点**：条件类型、分发条件类型、`[T] extends [string]` 阻止分发

#### 练习9：函数重载与类型推断

```typescript
// 实现一个重载函数 format：
// format(value: string): string  → 原样返回
// format(value: number): string  → 转为千分位字符串
// format(value: Date): string    → 转为 YYYY-MM-DD 格式

// 你的代码：

// 参考答案：
function format(value: string): string;
function format(value: number): string;
function format(value: Date): string;
function format(value: string | number | Date): string {
  if (typeof value === "string") {
    return value;
  }
  if (typeof value === "number") {
    return value.toLocaleString("en-US");
  }
  if (value instanceof Date) {
    return value.toISOString().split("T")[0];
  }
  throw new Error("Unsupported type");
}

// 测试
const s = format("hello");       // string
const n = format(1234567);       // string: "1,234,567"
const d = format(new Date());    // string: "2026-05-29"
```

**关键知识点**：函数重载、类型守卫、typeof / instanceof

### 第四部分：综合练习

#### 练习10：类型安全的 EventEmitter

```typescript
// 实现一个类型安全的事件系统：
// 1. 定义事件映射类型
// 2. on(event, callback) 注册事件
// 3. emit(event, data) 触发事件
// 4. off(event, callback) 移除事件
// 要求：事件名和数据类型严格绑定

// 参考答案：
interface EventMap {
  login: { userId: string };
  logout: { userId: string };
  error: { message: string; code: number };
}

class EventEmitter<Events extends Record<string, any>> {
  private listeners: {
    [K in keyof Events]?: Array<(data: Events[K]) => void>;
  } = {};

  on<K extends keyof Events>(event: K, callback: (data: Events[K]) => void): this {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(callback);
    return this;
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners[event]?.forEach(cb => cb(data));
  }

  off<K extends keyof Events>(event: K, callback: (data: Events[K]) => void): this {
    this.listeners[event] = this.listeners[event]?.filter(cb => cb !== callback);
    return this;
  }
}

// 测试
const emitter = new EventEmitter<EventMap>();

emitter
  .on("login", (data) => {
    console.log(`User ${data.userId} logged in`);
  })
  .on("error", (data) => {
    console.log(`Error ${data.code}: ${data.message}`);
  });

emitter.emit("login", { userId: "123" });
emitter.emit("error", { message: "Not Found", code: 404 });
// emitter.emit("login", { code: 404 });  // ❌ 数据类型不匹配
```

**关键知识点**：泛型 + keyof + 索引签名、链式调用、回调类型

## 实战案例

### 案例一：类型安全的表单验证

```typescript
// 用接口 + 泛型实现类型安全的表单验证
interface ValidationRule<T> {
  validator: (value: T) => boolean;
  message: string;
}

interface FieldConfig<T> {
  type: "text" | "number" | "email" | "password";
  label: string;
  required?: boolean;
  defaultValue?: T;
  rules?: ValidationRule<T>[];
}

type FormConfig<T extends Record<string, any>> = {
  [K in keyof T]: FieldConfig<T[K]>;
};

// 定义表单
interface LoginForm {
  email: string;
  password: string;
  remember: boolean;
}

const loginFormConfig: FormConfig<LoginForm> = {
  email: {
    type: "email",
    label: "邮箱",
    required: true,
    rules: [
      { validator: (v) => v.includes("@"), message: "邮箱格式不正确" },
    ],
  },
  password: {
    type: "password",
    label: "密码",
    required: true,
    rules: [
      { validator: (v) => v.length >= 8, message: "密码至少8位" },
    ],
  },
  remember: {
    type: "text",  // boolean 用 checkbox，这里简化
    label: "记住我",
    defaultValue: false,
  },
};

// 验证函数
function validateField<T>(
  value: T,
  rules?: ValidationRule<T>[]
): string[] {
  if (!rules) return [];
  return rules
    .filter(rule => !rule.validator(value))
    .map(rule => rule.message);
}
```

### 案例二：类型安全的 API Client

```typescript
// 泛型 + 接口实现类型安全的 API 客户端
interface ApiEndpoints {
  "/users": {
    GET: { response: User[]; query?: { active?: boolean } };
    POST: { response: User; body: Omit<User, "id"> };
  };
  "/users/:id": {
    GET: { response: User; params: { id: number } };
    PUT: { response: User; params: { id: number }; body: Partial<User> };
    DELETE: { response: void; params: { id: number } };
  };
}

type Method = "GET" | "POST" | "PUT" | "DELETE";

async function api<
  Path extends keyof ApiEndpoints,
  M extends keyof ApiEndpoints[Path] & Method
>(
  path: Path,
  method: M,
  options?: {
    params?: ApiEndpoints[Path][M] extends { params: infer P } ? P : never;
    query?: ApiEndpoints[Path][M] extends { query: infer Q } ? Q : never;
    body?: ApiEndpoints[Path][M] extends { body: infer B } ? B : never;
  }
): Promise<ApiEndpoints[Path][M] extends { response: infer R } ? R : never> {
  // 实现略...
  return {} as any;
}

// 类型安全的使用
const users = await api("/users", "GET");                    // User[]
const user = await api("/users/:id", "GET", { params: { id: 1 } });  // User
const created = await api("/users", "POST", { body: { name: "A", email: "a@b.c" } });  // User
```

## 底层原理

### 类型推断的核心算法

TypeScript 的类型推断基于**统一化算法（Unification）**：

```typescript
function identity<T>(value: T): T { return value; }

// 调用 identity("hello") 时的推断过程：
// 1. 参数 value 的声明类型是 T
// 2. 传入的实际类型是 string
// 3. 统一化：T = string
// 4. 返回值类型 T 被解析为 string

// 多参数推断
function merge<T, U>(a: T, b: U): T & U { return { ...a, ...b } as T & U; }
// 调用 merge({ x: 1 }, { y: "hello" })
// 1. a 的类型 { x: number } → T = { x: number }
// 2. b 的类型 { y: string } → U = { y: string }
// 3. 返回值 T & U = { x: number } & { y: string }
```

💡 **人话总结**：
- 类型练习的核心是**建立类型思维**：先想类型，再写代码
- 接口描述"是什么"，泛型描述"用什么"，类型推断"自动猜"
- 多写多练是唯一的捷径，没有银弹

## 高频面试题解析

### Q1：如何让一个函数的参数类型和返回值类型建立关联？

**参考答案：**
使用泛型参数建立类型关联：

```typescript
// ❌ 参数和返回值没有关联
function process(value: any): any { return value; }

// ✅ 泛型建立关联
function process<T>(value: T): T { return value; }

// 更复杂的关联
function transform<T, R>(value: T, fn: (val: T) => R): R {
  return fn(value);
}
```

### Q2：keyof、typeof、extends 在类型编程中分别起什么作用？

**参考答案：**
- **keyof**：取类型的所有键，生成联合类型
- **typeof**：取值的类型，在类型空间使用
- **extends**：类型约束或条件判断

```typescript
const person = { name: "Alice", age: 30 } as const;

type PersonKeys = keyof typeof person;    // "name" | "age"
type PersonType = typeof person;           // { readonly name: "Alice"; readonly age: 30 }

type IsPerson<T> = T extends PersonType ? "yes" : "no";
```

### Q3：如何实现一个类型安全的 Object.keys？

**参考答案：**
```typescript
// Object.keys 返回 string[]，丢失了键的类型信息
// 自定义类型安全版本
function typedKeys<T extends object>(obj: T): (keyof T)[] {
  return Object.keys(obj) as (keyof T)[];
}

const user = { name: "Alice", age: 30, active: true };
const keys = typedKeys(user);  // ("name" | "age" | "active")[]
```

### Q4：interface 和 type 在实际使用中如何选择？

**参考答案：**
```typescript
// 用 interface 的场景：
// 1. 定义对象的形状（可以被 extends 继承）
// 2. 需要声明合并（第三方库类型扩展）
// 3. 类的 implements

// 用 type 的场景：
// 1. 联合类型、交叉类型、条件类型
// 2. 映射类型、工具类型
// 3. 需要计算/推导的类型

// 经验法则：能 interface 就 interface，复杂的用 type
```

## 总结与扩展

### 核心要点

1. **接口定义**：readonly、可选、索引签名、继承是接口四大核心
2. **泛型使用**：函数/类/接口 + 约束 + 推断，构建类型安全的抽象
3. **类型推断**：理解 TypeScript 自动推断规则，减少冗余标注
4. **综合运用**：keyof + extends + 泛型 = 类型安全的核心公式

### 延伸学习方向

- **条件类型**：下一周的主题，泛型 + 条件判断
- **映射类型**：批量转换类型的利器
- **类型体操**：type-challenges 仓库的高级题目
- **声明文件**：为 JavaScript 库编写类型定义

### 相关主题

- [TypeScript基础类型系统](/2026/05/TypeScript基础类型系统/)
- [接口与类型别名的使用](/2026/05/接口与类型别名的使用/)
- [泛型基础与应用场景](/2026/05/泛型基础与应用场景/)
- [枚举的本质与编译结果](/2026/05/枚举的本质与编译结果/)
