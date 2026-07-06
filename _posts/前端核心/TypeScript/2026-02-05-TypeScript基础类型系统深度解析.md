---
layout: post
title: "TypeScript 基础类型系统"
date: 2026-02-05
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型系统", "基础类型", "any", "unknown", "never"]
---

## 一句话概括

TypeScript 在 JS 之上加了**静态类型层**，覆盖 string、number、boolean、array、tuple、enum、any、unknown、void、never 等基础类型——核心不是背概念，而是理解 `any` vs `unknown` 的安全取舍和 `never` 的穷尽检查价值。

## 核心知识点

### 1. 基础类型速览

```typescript
// 基本类型 —— 和 JS 一一对应
let name: string = 'TS';
let age: number = 30;
let done: boolean = false;

// 数组 —— 两种写法，推荐第一种
let nums: number[] = [1, 2, 3];
let strs: Array<string> = ['a', 'b'];

// 元组 —— 固定长度和类型，每个位置类型可以不同
let user: [string, number] = ['Alice', 30];
user[0].toUpperCase(); // ✅ TS 知道 [0] 是 string
// user[2];            // ❌ 越界报错

// 枚举
enum Dir { Up = 'UP', Down = 'DOWN' }
const d: Dir = Dir.Up;
```

### 2. any vs unknown — 类型安全的岔路口

```typescript
// any：完全放弃检查，自求多福
let a: any = 'hello';
a = 42;                  // ✅
a.nonExistentMethod();   // ✅ 编译通过，运行崩溃
let s1: string = a;      // ✅ any 可以赋值给任何类型

// unknown：类型安全的 any —— 必须先收窄
let u: unknown = 'hello';
u = 42;                  // ✅ 可以接收任何值
// u.toFixed();          // ❌ 不能直接使用
// let s2: string = u;   // ❌ 不能直接赋值

// ✅ 必须收窄后才能用
if (typeof u === 'number') {
  u.toFixed(2);          // OK，这里 u 是 number
  let n: number = u;     // OK
}
```

**铁律：** 不知道类型的时候用 `unknown` 而不是 `any`。`unknown` 逼你在使用前做类型守卫，`any` 是闭眼跳悬崖。

### 3. void vs never — 无返回值 vs 永远不返回

```typescript
// void：函数没有返回值（或返回 undefined）
function log(msg: string): void {
  console.log(msg);
}

// never：函数永远不会正常结束
function fail(msg: string): never {
  throw new Error(msg);       // 抛异常
}
function loop(): never {
  while (true) {}             // 死循环
}

// never 的杀手应用：穷尽检查
type Shape = 'circle' | 'square';
function area(s: Shape): number {
  switch (s) {
    case 'circle': return Math.PI;
    case 'square': return 1;
    default: {
      // 如果 Shape 新增了 'triangle'，这里 s 不再是 never，编译报错
      const _exhaustive: never = s;
      return _exhaustive;
    }
  }
}
```

### 4. null / undefined 与 strictNullChecks

```typescript
// strictNullChecks 关闭时（不推荐）：null/undefined 可以赋给任何类型
// let name: string = null;  // ⚠️ 通过，但不安全

// strictNullChecks 开启时（强烈推荐）：
let name: string = null;         // ❌
let name2: string | null = null; // ✅ 显式联合

// 可选属性 = 自动加 | undefined
interface User {
  name: string;
  age?: number;  // 等价于 age: number | undefined
}
```

### 5. 类型推断 vs 显式注解

```typescript
// ✅ 推断够了 —— 不用多写
let n = 42;           // number
let s = 'hello';      // string
let arr = [1, 2, 3];  // number[]

// ✅ 必须显式注解 —— 函数参数
function add(a: number, b: number): number {
  return a + b;
}

// 原则：能推断的别写，函数签名必须写
```

## 其实你每天都在用

- **API 返回值类型定义**：`interface ApiResponse<T> { code: number; data: T }` 让后端返回的数据结构一目了然
- **React useState 的类型推断**：`const [count, setCount] = useState(0)` 自动推断 `number`
- **表单校验**：`function validate(val: unknown): boolean` 用 unknown 接用户输入，再通过 typeof 收窄
- **事件处理**：`onChange={(e: React.ChangeEvent<HTMLInputElement>) => ...}` 精确到具体元素类型
- **配置常量枚举**：`enum Env { Dev, Prod }` 避免代码中散布魔法字符串 `'dev' / 'prod'`

## 常见误解

- **❌ 误区：「TS 就是给 JS 加冒号」** TypeScript 的类型系统远比"加冒号"复杂——它支持字面量类型、联合类型、条件类型、模板字面量类型等。基础类型只是入门第一课，真正的力量在高级类型体操里。

- **❌ 误区：「any 用着方便，先 any 后补类型」** 这是最常见的 TS 反模式。`any` 有"传染性"——一个 any 变量传给函数后，返回值也变 any，类型安全雪崩式丢失。正确做法是用 `unknown` 或显式写 `// TODO: type this` 标记。

- **❌ 误区：「void 和 undefined 是一回事」** 作为返回值类型：`void` 表示"我不关心返回值"（回调即使 return 了东西也被忽略）；`undefined` 表示"必须显式 return undefined"。作为变量类型：`void` 变量只能赋值 `undefined`（strict 下），本质区别在于语义约束。

- **❌ 误区：「never 就是个理论概念，实际用不上」** `never` 最实用的场景是**穷尽检查**（exhaustive check）。当你的联合类型新增了一个成员，如果所有处理分支都覆盖了，default 分支里的变量会被 TS 收窄为 `never`——一旦新增类型没覆盖，default 分支的类型就不是 `never` 了，编译直接报错。

## 一句话总结

TypeScript 基础类型的核心不是记住 12 种类型名，而是理解 three type safety pillars：用 `unknown` 替代 `any`、用 `strictNullChecks` 显式化空值、用 `never` 做穷尽检查——类型系统的安全等级由你选择。
