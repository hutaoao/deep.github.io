---
layout: post
title: "TypeScript 基础类型系统"
date: 2026-02-05
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型系统", "any", "unknown", "never", "面试"]
---

## 一句话概括

TypeScript 基础类型不是背 `string | number | boolean`——核心是三个安全支柱：用 `unknown` 锁死 any、用 `strictNullChecks` 显式化空值、用 `never` 做穷尽检查——选哪条路决定了你代码的类型安全等级。

## 核心知识点

### 1. 基础类型速览

```typescript
// 基本类型
let name: string = 'TS';
let age: number = 30;
let done: boolean = false;

// 数组（推荐第一种写法）
let nums: number[] = [1, 2, 3];

// 元组：固定长度 + 每位置类型独立
let user: [string, number] = ['Alice', 30];
user[0].toUpperCase(); // ✅ TS 知道 [0] 是 string

// 枚举
enum Status { Pending = 'PENDING', Success = 'SUCCESS' }
```

### 2. any vs unknown — 天差地别

```typescript
// any：放弃全部检查，等于回退到 JS
let a: any = 'hello';
a.nonExistentMethod(); // ✅ 编译通过，运行 💥
let s: string = a;     // ✅ any 污染一切

// unknown：安全的 any —— 必须收窄才能用
let u: unknown = 'hello';
// u.toUpperCase();    // ❌ 不能直接用
// let s: string = u;  // ❌ 不能直接赋值

// ✅ 收窄后使用
if (typeof u === 'string') {
  u.toUpperCase();           // OK
  const s: string = u;       // OK
}

// unknown 的最佳实践：API 返回值先用 unknown 接
const data: unknown = await fetch('/api').then(r => r.json());
if (typeof data === 'object' && data !== null && 'code' in data) {
  // 这里 data 被收窄为你预期的形状
}
```

**铁律：** 不知道类型的时候用 `unknown` 而不是 `any`。`unknown` 逼你在使用前做类型守卫，`any` 是闭眼跳悬崖。

### 3. void vs never — 无返回 vs 永远不返回

```typescript
// void：函数不返回有意义的值（或返回 undefined）
function log(msg: string): void { console.log(msg); }

// never：函数永远不会正常结束
function throwErr(msg: string): never { throw new Error(msg); }
function infiniteLoop(): never { while (true) {} }

// never 的杀手级应用：穷尽检查
type Shape = 'circle' | 'square';
function area(s: Shape): number {
  switch (s) {
    case 'circle': return Math.PI;
    case 'square': return 1;
    default: {
      // 如果 Shape 新增 'triangle'，s 的类型变成 'triangle'，不是 never → 编译报错
      const _exhaustive: never = s;
      return _exhaustive;
    }
  }
}
```

`never` 的穷尽检查是 TS 最被低估的功能——联合类型新增成员时，编译期就能发现问题。

### 4. strictNullChecks — 必开的选项

```typescript
// strictNullChecks: false（不推荐）
let name: string = null; // ✅ 通过，但不安全

// strictNullChecks: true（推荐）
let name: string = null;  // ❌ 类型错误
let name2: string | null = null; // ✅ 显式联合类型

// 可选属性自动带 undefined
interface User {
  name: string;
  age?: number; // 等价于 age: number | undefined
}
```

所有新项目都应该开 `strict: true`，里面包含了 `strictNullChecks`。

### 5. 类型推断原则

```typescript
// ✅ 能推断的就别写
let count = 0;        // TS 推断 number
let names = ['a'];    // TS 推断 string[]

// ✅ 必须显式注解的地方：函数参数
function add(a: number, b: number): number {
  return a + b;
}

// ✅ 对象字面量最好标类型（方便后续重构）
interface User { name: string; age: number }
const user: User = { name: 'Alice', age: 30 };
```

## 其实你每天都在用

- **API 响应类型**：`interface ApiRes<T> { code: number; data: T }` — 后端返回什么结构一目了然
- **useState 类型推断**：`const [count, setCount] = useState(0)` — TS 自动推断 `number`
- **表单输入收窄**：`function validate(val: unknown): val is string` — 用类型守卫从 unknown 收窄
- **事件处理**：`onChange={(e: ChangeEvent<HTMLInputElement>) => ...}` — 精确到元素类型
- **配置常量枚举**：`enum Env { Dev, Prod }` — 消灭魔法字符串

## 常见误解

- **❌ 误区：「TS 就是 JS + 冒号」** TypeScript 的类型系统有字面量类型、联合/交叉/条件/映射类型、递归类型——基础类型只是入门第一课。真正的力量在类型体操里。

- **❌ 误区：「any 方便，先 any 后面再补」** any 有**传染性**——一个 any 变量传给函数后，返回值也变 any，类型安全雪崩式扩散。正确做法：用 `unknown`，或显式写 `// @ts-expect-error 待定`。

- **❌ 误区：「void 和 undefined 一回事」** 当返回值类型时：`void` 表示"我不关心返回值"（即使回调里 return 了东西也被忽略）；`undefined` 表示"必须显式 return undefined"。当变量类型时：`void` 变量只能赋值 `undefined`（strict 下），但语义上表示"不应该被使用"。

- **❌ 误区：「never 是理论概念，实际用不上」** never 最实际的用途是穷尽检查——联合类型加新成员但没覆盖所有分支时编译直接报错。这是类型系统给你的免费 bug 检测器。

## 一句话总结

TypeScript 的三道安全锁：`unknown` 堵住 any 的口子，`strictNullChecks` 让空值无处隐藏，`never` 替你做穷尽检查——类型系统的安全感，来自你的选择。
