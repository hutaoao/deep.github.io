---
layout: post
title: "TypeScript 基础类型系统"
date: 2026-02-05
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型系统", "any", "unknown", "never", "面试"]
---

## 一句话概括

TypeScript 基础类型的面试不考你知不知道 `string | number`——考的是三道安全心法：为什么 `unknown` 碾压 `any`、`strictNullChecks` 到底防住了什么、`never` 凭什么是最被低估的类型。

## 核心知识点

### 1. 基础类型速览

```ts
// 原始类型——TS 能推断的别写
let name: string = 'TS';
let age = 30;                  // TS 自动推断 number
let done = false;              // TS 自动推断 boolean

// 数组：推荐 T[] 写法，简洁统一
const nums: number[] = [1, 2, 3];

// 元组：固定长度 + 每位置类型独立
const user: [string, number] = ['Alice', 30];
user[0].toUpperCase(); // ✅ TS 知道索引 0 是 string

// 枚举：字符串枚举推荐（可读性好，不生成反向映射）
enum Direction { Up = 'UP', Down = 'DOWN' }

// 字面量类型——精确到值的类型
let axis: 'x' | 'y' = 'x';
let dice: 1 | 2 | 3 | 4 | 5 | 6 = 3;
```

### 2. any vs unknown — 甩锅 vs 锁门

```ts
// ❌ any：放弃一切类型检查，等于玩火
let a: any = 'hello';
a.nonExistent();    // 编译通过，运行时 💥
let s: string = a;  // any 传染给 s——整条链的类型检查废了

// ✅ unknown：必须收窄才能用，类型安全
let u: unknown = 'hello';
// u.toUpperCase();   // ❌ 编译报错——TS 不知道它是 string
if (typeof u === 'string') {
  u.toUpperCase();    // ✅ 收窄后安全
}

// 最佳实践：API 返回值用 unknown 接，然后类型守卫收窄
const raw: unknown = await fetch('/api/user').then(r => r.json());
// 不校验就用 → 编译报错；校验后 → 类型安全
```

**铁律：** 不确定类型用 `unknown`——它逼你做类型守卫。`any` 是甩手掌柜——不光自己裸奔，还感染所有跟它沾边的变量。ESLint 配 `no-explicit-any` 是最低安全基准。

### 3. void vs never — 别混为一谈

```ts
// void：函数不返回有意义的值（返回了也不关心）
function log(msg: string): void { console.log(msg); }

// never：函数永远不会正常结束
function throwErr(msg: string): never { throw new Error(msg); }
function infiniteLoop(): never { while (true) {} }

// 🔥 never 的杀手级应用：穷尽检查
type Shape = 'circle' | 'square' | 'triangle';

function area(s: Shape): number {
  switch (s) {
    case 'circle':   return Math.PI;
    case 'square':   return 1;
    case 'triangle': return 0.5;
    default: {
      // 所有 case 覆盖完 → s 类型为 never → 编译通过
      // 未来加 'rectangle' → s 变 'rectangle' → 编译报错！
      const _exhaustive: never = s;
      return _exhaustive;
    }
  }
}
```

### 4. strictNullChecks — 第一道防线

```ts
// strictNullChecks: false（自掘坟墓）
let name: string = null;  // 编译通过，运行时必炸

// strictNullChecks: true（所有新项目必须开）
let name: string = null;         // ❌ 编译报错
let name2: string | null = null; // ✅ 显式声明，null 无处躲藏

// 可选属性的真相：
interface User {
  name: string;
  age?: number;   // 等价于 age: number | undefined
}
```

**直接开 `"strict": true`**——它包含 `strictNullChecks` + `strictFunctionTypes` + 四个子选项。新项目不开 strict 等于花钱请保镖却不让他带武器。

### 5. 何时写类型注解

```ts
// ✅ 能推断的别写
let count = 0;               // TS 推断 number
const names = ['a', 'b'];    // TS 推断 string[]

// ✅ 必须写：函数参数
function add(a: number, b: number): number { return a + b; }

// ✅ 建议写：对象字面量（便于重构和 IntelliSense）
interface User { name: string; age: number }
const user: User = { name: 'Alice', age: 30 };
```

**原则：** 让 TS 做推断，你在**边界**写类型——函数签名、API 接口、导出的公共方法。类型注解是给"读者"看的，不是给编译器看的。

## 其实你每天都在用

- **`useState` 自动推断**：`const [count, setCount] = useState(0)` → TS 自动 `number`
- **API 响应泛型**：`interface ApiRes<T> { code: number; data: T }` → 一份定义复用所有接口
- **事件处理精确类型**：`onClick={(e: React.MouseEvent) => {}}` → `e.target` 类型精确到具体 DOM 元素
- **`never` 穷尽检查**：Redux reducer 的 `default: const _: never = action` → 漏 case 编译报，CI 直接拦截
- **类型守卫收窄**：`if (typeof input === 'string') { input.toLowerCase() }` → TS 自动收窄，无需 `as`

## 常见误解（FAQ）

- **❌ 误区：「any 方便，之后再补类型」** any 有**传染性**——`fn(anyValue)` 的返回值也变 any，污染雪崩式扩散。正确姿势：用 `unknown` + 类型守卫，或用 `// @ts-expect-error` 明确标注临时绕过（带注释说明原因）。

- **❌ 误区：「TypeScript 就是 JS + 冒号类型注解」** 基础类型只是冰山一角。TS 的类型系统有泛型、条件类型、映射类型、模板字面量类型、递归类型——它的表达能力接近一门完整的类型编程语言。只用基础类型等于拿跑车买菜。

- **❌ 误区：「void 和 undefined 是一回事」** 返回值语义不同：`void` 表示"不关心返回值"（回调里 return 了也会被忽略），`undefined` 表示"必须返回或隐式返回 undefined"。运行时 `void` 函数的返回值类型系统会忽略。

- **❌ 误区：「never 太抽象，实际用不到」** `never` 的穷尽检查是类型系统给你的**免费 bug 检测器**。联合类型加新成员 → 忘加新 case → 编译直接报错，CI 阶段就拦截，不用等到线上炸。Redux 社区里这是标准模式。

## 一句话总结

TypeScript 三道安全锁：`unknown` 堵 any 的口子、`strictNullChecks` 把空值显式化、`never` 替你穷尽检查。类型安全的等级不取决于你知道多少类型，而在于你选择了哪条防线。
