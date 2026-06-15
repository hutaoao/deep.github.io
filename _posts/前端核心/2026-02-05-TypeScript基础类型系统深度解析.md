---
title: "TypeScript基础类型系统深度解析"
date: 2026-02-05
categories: [前端核心, TypeScript]
tags: [TypeScript, 类型系统, 静态类型, 类型推断, 基础类型]
---

## 📌 一句话概括

TypeScript 的类型系统在 JavaScript 的动态类型之上引入了静态类型注解，覆盖 string、number、boolean、array、tuple、enum、any、unknown、void、never 等基础类型，在编译期捕获错误、提升代码可维护性。

## 背景

JavaScript 是一门动态弱类型语言——变量在运行时可以持有任何类型的值，类型错误往往在代码执行到那一行时才暴露。这在小型脚本中问题不大，但当中大型项目代码量增长后，动态类型带来的隐患日益突出：

- **运行时才发现类型错误**：`undefined is not a function`、`Cannot read property of null` 等错误无法提前感知
- **重构困难**：修改一个函数签名后，所有调用处都要靠人工确认
- **IDE 补全受限**：编辑器无法确定变量类型，自动补全和重构能力大打折扣

TypeScript 由 Microsoft 于 2012 年发布，目标是在 JavaScript 的基础上增加**可选的静态类型系统**。它并不创造新的运行时类型——所有类型注解在编译后都会被擦除，但它让开发者在编码阶段就能获得类型安全保障。

**TypeScript 类型系统的设计哲学：**

1. **渐进式类型化**：可以从 JavaScript 逐步迁移，不必一步到位
2. **结构化类型（Structural Typing）**：类型兼容性基于结构而非名义
3. **类型推断**：能推断的地方不需要手写注解
4. **类型擦除**：编译输出纯 JavaScript，零运行时开销

## 概念与定义

### 基础类型总览

| 类型 | 说明 | 示例 |
|------|------|------|
| `string` | 字符串 | `let name: string = "ts"` |
| `number` | 数值（整数、浮点、二/八/十六进制） | `let age: number = 25` |
| `boolean` | 布尔值 | `let done: boolean = true` |
| `array` | 数组 | `let list: number[] = [1, 2]` |
| `tuple` | 元组（固定长度与类型的数组） | `let x: [string, number]` |
| `enum` | 枚举 | `enum Color { Red, Green }` |
| `any` | 任意类型（绕过检查） | `let x: any = 42` |
| `unknown` | 安全的 any | `let x: unknown = 42` |
| `void` | 无返回值 | `function log(): void {}` |
| `never` | 永不存在的值 | `function fail(): never { throw }` |
| `null` / `undefined` | 空值 | `let x: null = null` |
| `object` | 非原始类型 | `let o: object = {}` |

### 类型注解 vs 类型推断

```typescript
// 类型注解：显式声明变量类型
let name: string = "TypeScript";

// 类型推断：编译器自动推断类型
let age = 25;           // 推断为 number
let message = "hello";  // 推断为 string
```

**原则：能在初始化时推断出的类型，不需要手动注解；函数参数和返回值建议显式注解。**

## 最小示例

```typescript
// 基础类型声明
let username: string = "Alice";
let age: number = 30;
let isActive: boolean = true;

// 数组两种写法
let scores: number[] = [95, 87, 92];
let names: Array<string> = ["Alice", "Bob"];

// 元组
let user: [string, number] = ["Alice", 30];

// 枚举
enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}
let dir: Direction = Direction.Up;

// any 与 unknown
let anything: any = "hello";
anything = 42;         // OK，any 绕过检查

let safe: unknown = "hello";
safe = 42;             // OK，unknown 可以接收任何值
// safe.toFixed();     // ❌ Error: unknown 类型不能直接使用

// void 与 never
function log(msg: string): void {
  console.log(msg);
}

function throwError(msg: string): never {
  throw new Error(msg);
}
```

## 核心知识点拆解

### 1. string — 字符串类型

TypeScript 的 `string` 对应 JavaScript 的原始字符串类型，与 `String`（包装对象）不同：

```typescript
let name: string = "TypeScript";
// let name2: String = "TS";  // 不推荐，String 是包装对象类型

// 模板字符串也是 string
let greeting: string = `Hello, ${name}`;

// 字符串字面量类型（高级用法）
type EventName = "click" | "hover" | "focus";
let event: EventName = "click";  // 只能是这三个值之一
```

### 2. number — 数值类型

JavaScript 中所有数字都是浮点数（IEEE 754 双精度），TypeScript 的 `number` 也一样：

```typescript
let decimal: number = 6;
let hex: number = 0xf00d;
let binary: number = 0b1010;
let octal: number = 0o744;

// 大整数用 bigint（ES2020）
let big: bigint = 100n;
```

**注意：** `number` 和 `bigint` 是不同的类型，不能互相赋值。

### 3. boolean — 布尔类型

```typescript
let isDone: boolean = false;

// 构造函数 Boolean 返回的是 Boolean 对象，不是 boolean
// let b: boolean = new Boolean(true);  // ❌ Error
let b: boolean = Boolean(true);         // ✅ OK，调用函数
```

### 4. array — 数组类型

两种等价写法：

```typescript
// 写法一：元素类型[]
let numbers: number[] = [1, 2, 3];

// 写法二：泛型数组 Array<元素类型>
let strings: Array<string> = ["a", "b", "c"];

// 推荐写法一，更简洁
// 多维数组
let matrix: number[][] = [[1, 2], [3, 4]];
```

**只读数组：**

```typescript
let readOnlyArr: readonly number[] = [1, 2, 3];
// readOnlyArr.push(4);  // ❌ Error: readonly 数组不能修改

// 等价写法
let readOnlyArr2: ReadonlyArray<number> = [1, 2, 3];
```

### 5. tuple — 元组类型

元组是**固定长度、固定类型**的数组，每个位置的类型可以不同：

```typescript
// 定义元组
let user: [string, number] = ["Alice", 30];

// 访问元素——类型安全
user[0].toUpperCase();  // ✅ TypeScript 知道 user[0] 是 string
user[1].toFixed(2);     // ✅ TypeScript 知道 user[1] 是 number

// 越界访问
// user[2];  // ❌ Error: 没有索引 2

// 可选元素
let point: [number, number, number?] = [1, 2];  // 第三维可选

// 命名元组（提高可读性）
let namedTuple: [name: string, age: number] = ["Alice", 30];
```

**元组 vs 数组：**

| 特性 | 数组 | 元组 |
|------|------|------|
| 长度 | 不固定 | 固定（除非可选元素） |
| 元素类型 | 统一 | 每位可不同 |
| push | 允许 | 允许（但视为联合类型） |
| 越界访问 | undefined | 编译报错 |

### 6. enum — 枚举类型

枚举为一组命名常量提供友好名称：

```typescript
// 数字枚举（默认从 0 递增）
enum Direction {
  Up,      // 0
  Down,    // 1
  Left,    // 2
  Right,   // 3
}

// 反向映射：数字枚举支持 值→名称
console.log(Direction[0]);  // "Up"
console.log(Direction.Up);  // 0

// 字符串枚举（无反向映射）
enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
  Pending = "PENDING",
}

// 异构枚举（不推荐：数字和字符串混用）
enum BooleanLikeHeterogeneousEnum {
  No = 0,
  Yes = "YES",
}

// const enum（编译时内联，不生成运行时代码）
const enum Color {
  Red,
  Green,
  Blue,
}
let c = Color.Red;  // 编译为 let c = 0 /* Color.Red */
```

**枚举的注意事项：**
- 数字枚举存在**反向映射**，字符串枚举不存在
- 数字枚举可以不初始化（自动递增），字符串枚举必须显式赋值
- `const enum` 在编译时被内联替换，不产生额外 JS 代码
- `--isolatedModules` 模式下，`const enum` 有使用限制

### 7. any — 逃逸类型

`any` 是 TypeScript 的"后门"——完全放弃类型检查：

```typescript
let x: any = "hello";
x = 42;               // OK
x.toFixed();           // OK（但运行时可能报错）
x.nonExistentMethod(); // OK（编译通过，运行时崩溃）

// 隐式 any：未注解且无法推断时
function fn(param) { }  // param 隐式 any（strict 模式下报错）
```

**使用场景：**
- 渐进式迁移 JavaScript 代码时临时使用
- 与第三方库交互，类型定义不完善时
- **应尽量缩小 `any` 的范围，避免污染传播**

```typescript
// ❌ any 污染：返回值也是 any，类型安全全丢
function parseJSON(str: string): any {
  return JSON.parse(str);
}

// ✅ 用 unknown 替代，使用时必须收窄
function parseJSON(str: string): unknown {
  return JSON.parse(str);
}
```

### 8. unknown — 类型安全的 any

`unknown` 是 TypeScript 3.0 引入的**顶层类型（Top Type）**：

```typescript
let value: unknown;

value = "hello";    // OK
value = 42;         // OK
value = { a: 1 };   // OK

// ❌ 不能直接使用 unknown 类型的值
// value.toFixed();        // Error
// value.toUpperCase();    // Error
// value * 2;              // Error

// ✅ 必须先收窄类型
if (typeof value === "number") {
  value.toFixed();   // OK，收窄为 number
}

// 类型守卫收窄
function formatValue(value: unknown): string {
  if (typeof value === "string") return value.toUpperCase();
  if (typeof value === "number") return value.toFixed(2);
  if (Array.isArray(value)) return value.join(", ");
  return String(value);
}
```

**any vs unknown 对比：**

| 特性 | any | unknown |
|------|-----|---------|
| 接收任何值 | ✅ | ✅ |
| 赋值给其他类型 | ✅（直接通过） | ❌（必须收窄） |
| 直接使用属性/方法 | ✅（不检查） | ❌（必须收窄） |
| 类型安全 | ❌ | ✅ |

### 9. void — 无返回值

`void` 表示函数没有返回值（或返回 `undefined`）：

```typescript
function log(msg: string): void {
  console.log(msg);
  // 没有 return，或 return;
}

// void 变量的实际意义不大
let unusable: void = undefined;
```

**void 在回调类型中的使用：**

```typescript
type Callback = (data: string) => void;

function fetchData(cb: Callback) {
  cb("result");
}

// 返回值被忽略，即使实际返回了值
fetchData((data) => {
  console.log(data);
  return 42;  // 不会报错，因为 void 回调允许返回值
});
```

### 10. never — 永不存在的类型

`never` 是类型系统的**底层类型（Bottom Type）**，表示永远不会有值：

```typescript
// 场景一：抛出异常的函数
function fail(msg: string): never {
  throw new Error(msg);
}

// 场景二：无限循环
function infiniteLoop(): never {
  while (true) {}
}

// 场景三：类型收窄的穷尽检查
type Shape = "circle" | "square";

function getArea(shape: Shape): number {
  switch (shape) {
    case "circle": return Math.PI;
    case "square": return 1;
    default:
      // 如果所有分支都处理了，shape 在这里是 never
      const _exhaustiveCheck: never = shape;
      return _exhaustiveCheck;
  }
}

// 场景四：交叉类型不可能存在
type Impossible = string & number;  // never
```

**never 的特性：**
- `never` 是所有类型的子类型（可以赋值给任何类型）
- 没有任何类型是 `never` 的子类型（除了 `never` 本身）
- 在联合类型中，`never` 会被消除：`string | never` = `string`

### 11. null / undefined — 空值类型

```typescript
// 默认情况下 null 和 undefined 是所有类型的子类型
let num: number = undefined;  // ⚠️ strictNullChecks 关闭时 OK

// 开启 strictNullChecks 后（推荐）
let num2: number = undefined;  // ❌ Error
let num3: number | undefined = undefined;  // ✅

// 可选属性和参数自动添加 | undefined
interface User {
  name: string;
  age?: number;  // 等价于 age: number | undefined
}
```

**最佳实践：始终开启 `strictNullChecks`，让 null 和 undefined 显式化。**

## 实战案例

### 案例1：API 响应类型定义

```typescript
// 定义后端返回数据的类型结构
interface ApiResponse<T> {
  code: number;
  message: string;
  data: T;
}

interface User {
  id: number;
  name: string;
  email: string;
  role: "admin" | "user" | "guest";  // 字符串字面量联合
}

// 使用
function fetchUser(id: number): ApiResponse<User> {
  // 模拟
  return {
    code: 200,
    message: "success",
    data: {
      id,
      name: "Alice",
      email: "alice@example.com",
      role: "admin",
    },
  };
}

const res = fetchUser(1);
res.data.name.toUpperCase();  // ✅ 类型安全
```

### 案例2：表单验证中的联合类型

```typescript
// 使用联合类型定义表单字段类型
type FieldType = "text" | "number" | "email" | "password";

interface FormField {
  type: FieldType;
  label: string;
  required: boolean;
  placeholder?: string;        // 可选属性
  validator?: (value: unknown) => boolean;  // unknown 收窄
}

const fields: FormField[] = [
  {
    type: "text",
    label: "用户名",
    required: true,
    placeholder: "请输入用户名",
  },
  {
    type: "email",
    label: "邮箱",
    required: true,
    validator: (value: unknown): boolean => {
      if (typeof value !== "string") return false;
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
    },
  },
  {
    type: "number",
    label: "年龄",
    required: false,
  },
];
```

### 案例3：使用 unknown 安全解析 JSON

```typescript
function safeParseJSON(value: string): unknown {
  try {
    return JSON.parse(value);
  } catch {
    return undefined;
  }
}

// 类型守卫：判断是否为对象
function isObject(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

// 安全使用
const raw = '{"name": "Alice", "age": 30}';
const parsed = safeParseJSON(raw);

if (isObject(parsed)) {
  const name = parsed.name;   // unknown，还需进一步收窄
  if (typeof name === "string") {
    console.log(name.toUpperCase());  // ✅ 安全
  }
}
```

## 底层原理

### 1. 类型擦除（Type Erasure）

TypeScript 的类型注解在编译后完全消失，不会产生任何运行时代码：

```typescript
// 源码
let name: string = "Alice";
function add(a: number, b: number): number {
  return a + b;
}

// 编译输出
let name = "Alice";
function add(a, b) {
  return a + b;
}
```

这意味着：
- TypeScript 的类型检查**只在编译时生效**
- 运行时没有任何类型信息（除非手动保留）
- 类型错误不会阻止 JavaScript 执行（除非配置 `noEmitOnError`）

### 2. 结构化类型（Structural Typing）

TypeScript 采用**结构化类型系统**（也叫鸭子类型 duck typing）——只要结构兼容，类型就兼容：

```typescript
interface Named {
  name: string;
}

interface Person {
  name: string;
  age: number;
}

let person: Person = { name: "Alice", age: 30 };
let named: Named = person;  // ✅ Person 结构上满足 Named

// 对比：名义类型系统（如 Java）中，即使结构相同，类名不同也不兼容
```

### 3. 类型检查流程

TypeScript 编译器的工作流程：

1. **源码解析**：将 TypeScript 代码解析为 AST
2. **绑定**：建立符号表，关联声明与引用
3. **类型检查**：
   - 推断每个表达式的类型
   - 检查赋值兼容性
   - 检查函数调用参数类型
   - 检查属性访问是否存在
4. **类型擦除**：移除所有类型注解
5. **代码生成**：输出 JavaScript 代码

### 4. 类型兼容性规则

TypeScript 的类型兼容遵循以下原则：

```typescript
// 子类型可以赋值给父类型
let s: string = "hello";
let u: unknown = s;        // ✅ string 是 unknown 的子类型

// 函数参数类型是逆变的（默认启用 strictFunctionTypes）
type Fn1 = (x: string) => void;
type Fn2 = (x: string | number) => void;

let f1: Fn1 = (x) => console.log(x);
let f2: Fn2 = f1;  // ✅ Fn1 可以赋值给 Fn2（参数逆变）
```

## 高频面试题解析

### Q1：any 和 unknown 有什么区别？

**答：**

| 维度 | any | unknown |
|------|-----|---------|
| 类型安全 | 完全放弃检查 | 必须收窄后使用 |
| 赋值给其他类型 | 直接通过 | 必须收窄 |
| 顶层/底层类型 | 既是顶层又是底层 | 只是顶层类型 |
| 推荐程度 | 不推荐 | 推荐替代 any |

```typescript
let a: any = "hello";
let b: unknown = "hello";

let s1: string = a;   // ✅ any 可以赋值给任何类型
let s2: string = b;   // ❌ unknown 不能直接赋值

a.toUpperCase();       // ✅ 编译通过（运行时可能报错）
b.toUpperCase();       // ❌ 必须先收窄

if (typeof b === "string") {
  b.toUpperCase();     // ✅ 收窄后可使用
}
```

**结论：** `unknown` 是类型安全的 `any`，应优先使用。

### Q2：void 和 undefined 有什么区别？

**答：**

```typescript
// void 表示"不关心返回值"，undefined 是一个具体的类型
function logA(): void {
  console.log("A");
  // 没有 return
}

function logB(): undefined {
  console.log("B");
  return undefined;  // 必须显式 return
}

// void 变量只能赋值 undefined 或 null（strictNullChecks 关闭时）
let v: void = undefined;

// 函数类型赋值
type VoidFn = () => void;
type UndefinedFn = () => undefined;

const f1: VoidFn = () => 42;       // ✅ 返回值被忽略
const f2: UndefinedFn = () => 42;  // ❌ 返回值必须是 undefined
```

### Q3：never 类型在什么时候使用？

**答：** `never` 的四种典型场景：

```typescript
// 1. 抛异常
function assertNever(x: never): never {
  throw new Error("Unexpected value: " + x);
}

// 2. 穷尽检查（Exhaustive Check）
type Action = "increment" | "decrement";

function reducer(action: Action): number {
  switch (action) {
    case "increment": return 1;
    case "decrement": return -1;
    default:
      return assertNever(action);  // 如果 Action 增加新值，这里会报错
  }
}

// 3. 不可达代码
type StringOrNumber = string | number;
function check(val: StringOrNumber) {
  if (typeof val === "string") {
    // val: string
  } else if (typeof val === "number") {
    // val: number
  } else {
    // val: never（不可达）
  }
}

// 4. 条件类型中过滤
type NonNullable<T> = T extends null | undefined ? never : T;
type Result = NonNullable<string | null | undefined>;  // string
```

### Q4：tuple 和 array 的区别是什么？

**答：**

```typescript
// Array：长度不限，元素类型统一
let arr: number[] = [1, 2, 3];
arr.push(4);        // ✅
arr[100];           // number（实际可能是 undefined）

// Tuple：固定长度，每位类型可不同
let tup: [string, number] = ["age", 25];
tup.push("extra");  // ✅ 但不推荐，破坏语义
// tup[2];           // ❌ 编译报错（长度固定为 2）

// 命名元组提高可读性
let record: [key: string, value: number] = ["age", 25];

// 实际应用：React useState 返回值类型
// const [state, setState] = useState(initialValue);
// 类型为 [state: T, dispatch: Dispatch<T>]
```

### Q5：TypeScript 的类型推断规则是什么？

**答：** TypeScript 在以下场景会自动推断类型：

```typescript
// 1. 变量初始化推断
let x = 42;          // number
let s = "hello";     // string
let arr = [1, 2, 3]; // number[]

// 2. 函数返回值推断
function add(a: number, b: number) {
  return a + b;      // 推断返回 number
}

// 3. 上下文类型推断（回调参数）
const names = ["Alice", "Bob", "Charlie"];
names.map(name => name.toUpperCase());  // name 推断为 string

// 4. 最佳通用类型推断
let mixed = [1, "hello", true];  // (string | number | boolean)[]
```

**最佳实践：**
- 函数参数必须显式注解
- 简单变量可以依赖推断
- 复杂对象建议显式注解（提高可读性）

## 总结与扩展

### 核心要点

1. **TypeScript 基础类型是 JavaScript 类型的超集**
   - string / number / boolean 与 JS 一一对应
   - tuple / enum / unknown / never 是 TS 独有的

2. **any 是逃生舱，unknown 是安全替代**
   - 优先使用 unknown 代替 any
   - 使用 any 时尽量缩小作用范围

3. **never 是类型系统的底线**
   - 穷尽检查、不可达代码、条件类型过滤

4. **元组 vs 数组语义不同**
   - 数组：同类型、长度不限
   - 元组：异类型、长度固定

5. **类型推断减少冗余注解**
   - 能推断的不写，函数参数必须写

### 扩展学习

- **类型拓宽（Widening）与收窄（Narrowing）**：TypeScript 如何在赋值和使用时调整类型
- **字面量类型**：`type Direction = "left" | "right"` 的精确约束
- **联合类型与交叉类型**：组合类型的两种方式
- **类型守卫（Type Guard）**：如何安全收窄 unknown / 联合类型
- **泛型基础**：从基础类型到参数化类型的进阶

### 实战建议

1. **开启 strict 模式**：`strict: true` 是新项目的底线
2. **显式注解公共 API**：函数签名、接口定义必须写类型
3. **避免滥用 any**：用 unknown + 类型守卫替代
4. **善用枚举管理常量**：比魔法数字和字符串更安全
5. **利用类型推断**：不需要到处写注解，保持代码简洁

---

**参考资料：**
- TypeScript Handbook - Everyday Types
- TypeScript Deep Dive - Basarat Ali Syed
- Microsoft TypeScript 官方文档
- 《TypeScript 编程》- Boris Cherny
