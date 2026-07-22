---
layout: post
title: "TypeScript 基础类型系统"
date: 2026-02-05
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型系统", "any", "unknown", "never", "面试"]
---

## 一句话概括

TypeScript 基础类型的面试不考你知不知道 `string | number`——考的是三个安全心法：为什么用 `unknown` 而不是 `any`、`strictNullChecks` 到底防住了什么、`never` 凭什么是最被低估的类型。

## 核心知识点

### 1. 基础类型速览

```typescript
// 原始类型
let name: string = 'TS';
let age: number = 30;
let done: boolean = false;

// 数组（推荐 T[] 写法）
let nums: number[] = [1, 2, 3];
let matrix: number[][] = [[1, 2], [3, 4]];

// 元组：固定长度 + 每位置类型独立
let user: [string, number] = ['Alice', 30];
user[0].toUpperCase(); // ✅ TS 知道索引 0 是 string

// 枚举
enum Status { Pending, Success, Fail }       // 数字枚举（自增）
enum Direction { Up = 'UP', Down = 'DOWN' }  // 字符串枚举

// 字面量类型 —— 精确到值的类型
let direction: 'left' | 'right' = 'left';   // 只接受这两个字符串
let count: 1 | 2 | 3 = 2;                   // 只接受这三个数字
```

### 2. any vs unknown — 一个开放、一把锁

```typescript
// ❌ any：放弃所有类型检查，等于回到 JS
let a: any = 'hello';
a.nonExistentMethod();  // 编译通过，运行时 💥
let s: string = a;      // any 污染了 s——s 的类型检查也废了

// ✅ unknown：安全的 any——必须收窄才能用
let u: unknown = 'hello';
// u.toUpperCase();     // ❌ 不能直接用，TS 不知道 u 是什么
// let s: string = u;   // ❌ 不能直接赋值给 string

// 必须收窄：
if (typeof u === 'string') {
  u.toUpperCase();           // ✅ TS 知道这里是 string
  const s: string = u;       // ✅
}

// 最佳实践：API 返回值先用 unknown 接，校验完再断言
const raw: unknown = await fetch('/api').then(r => r.json());
if (typeof raw === 'object' && raw !== null && 'data' in raw) {
  // 这里 raw 的类型被 TS 收窄了
}
```

**铁律：** 不确定类型的时候用 `unknown`，它会逼你在使用前做类型守卫。`any` 是闭眼跳过所有安全检查——传染性极强，一个 `any` 变量能污染整条数据流。

### 3. void vs never — 别混为一谈

```typescript
// void：函数不返回有意义的值（或隐式返回 undefined）
function log(msg: string): void { console.log(msg); }
// 回调里 return 了东西也没事，void 表示"我不关心返回值"

// never：函数永远不会正常结束
function throwErr(msg: string): never { throw new Error(msg); }
function infiniteLoop(): never { while (true) {} }

// never 的杀手级应用：穷尽检查
type Shape = 'circle' | 'square' | 'triangle';
function area(s: Shape): number {
  switch (s) {
    case 'circle': return Math.PI;
    case 'square': return 1;
    case 'triangle': return 0.5;  // 忘了加 triangle？下面会报错
    default: {
      // s 的类型在这里是 never → 编译通过，说明所有 case 都覆盖了
      // 如果漏掉一个联合类型成员，s 就不是 never → 编译报错
      const _exhaustive: never = s;
      return _exhaustive;
    }
  }
}
```

### 4. strictNullChecks — 第一件要做的事

```typescript
// strictNullChecks: false（默认关，但强烈不推荐）
let name: string = null;  // ✅ 通过——但你 100% 会在运行时踩坑

// strictNullChecks: true（推荐）
let name: string = null;        // ❌ 编译报错
let name2: string | null = null; // ✅ 显式联合——null 被标出来了

// 可选属性背后发生了什么
interface User {
  name: string;
  age?: number;   // 等价于 age: number | undefined
}
```

**所有新项目请在 `tsconfig.json` 里写 `"strict": true`——** 它包含了 `strictNullChecks`、`strictFunctionTypes` 等六个子选项，是类型安全的基础设施。

### 5. 何时写类型注解

```typescript
// ✅ 能推断的就别写
let count = 0;            // TS 推断 number
let names = ['a', 'b'];   // TS 推断 string[]

// ✅ 必须写的地方：函数参数
function add(a: number, b: number): number { return a + b; }

// ✅ 建议写的地方：对象字面量（方便重构和 intellisense）
interface User { name: string; age: number }
const user: User = { name: 'Alice', age: 30 };
```

**原则：** 让 TS 帮你做尽可能多的推断，你在**边界**写类型——函数签名、API 接口、导出的公共方法。

## 其实你每天都在用

- **API 响应类型泛型化**：`interface ApiRes<T> { code: number; data: T }` —— 一份定义复用所有接口
- **useState 自动推断**：`const [count, setCount] = useState(0)` —— TS 自动推断 `number`
- **表单校验类型守卫**：`function isValid(val: unknown): val is User` —— 收窄 unknown 为具体类型
- **事件处理精确类型**：`onChange={(e: ChangeEvent<HTMLInputElement>) => ...}` —— 精确到 DOM 元素类型
- **never 穷尽检查**：Redux reducer 的 switch-case 里加 `default: const _: never = action` —— 漏了 case 编译直接报

## 常见误解

- **❌ 误区：「TypeScript 就是 JS + 冒号加类型」** 基础类型只是冰山上最薄的一层。TS 的类型系统有泛型、条件类型、映射类型、模板字面量类型、递归类型——它的表达能力接近一个完整的类型编程语言。

- **❌ 误区：「any 方便，之后有时间再补类型」** any 有**传染性**——`fn(anyValue)` 的返回值也变 any，类型安全雪崩式扩散。正确做法：用 `unknown`，或显式标 `// @ts-expect-error 原因`。

- **❌ 误区：「void 和 undefined 一回事」** 当**返回值**时：`void` 表示"调用方不应该关心返回了什么"（即使回调 return 了值也会被忽略）；`undefined` 表示"必须显式 return undefined 或没有 return 语句"。当**变量**时：`void` 类型变量只能赋值 `undefined`，但它语义上表示"这个值不应该被使用"。

- **❌ 误区：「never 太抽象，实际项目用不到」** never 最实际的用途——穷尽检查——是类型系统给你的**免费 bug 检测器**。联合类型加新成员 → 忘加新 case → 编译报错，在 CI 阶段直接拦截。

## 一句话总结

TypeScript 的三道安全锁：`unknown` 堵住 any 的口子、`strictNullChecks` 把空值显式化、`never` 替你穷尽检查——类型安全的等级，不取决于你知道多少类型，而取决于你选择了哪条防线。
