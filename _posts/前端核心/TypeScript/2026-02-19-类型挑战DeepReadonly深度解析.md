---
layout: post
title: "类型挑战：DeepReadonly"
date: 2026-02-19 00:00:00 +0800
categories: [前端核心, TypeScript]
tags: [TypeScript, 类型体操, DeepReadonly, 递归类型, 面试题]
---

> **一句话概括：** `DeepReadonly<T>` 是 TypeScript 类型体操中核心的"中等难度"题目，它要求将对象类型及其所有嵌套子对象的全部属性递归地标记为 `readonly`，是检验递归类型、映射类型和边界条件处理能力的黄金标准。

## 背景与意义

TypeScript 内置的 `Readonly<T>` 工具类型只能处理一层深度：

```typescript
interface Config {
  server: {
    host: string;
    port: number;
  };
  debug: boolean;
}

type ReadonlyConfig = Readonly<Config>;
// 等价于：
// {
//   readonly server: { host: string; port: number; };
//   readonly debug: boolean;
// }
// ↑ 注意：server 对象本身的内容仍然是可变的！
```

在实际项目中，仅仅浅层只读远远不够。你可能会遇到以下场景：

- **Immutable 状态管理**：确保整个状态树完全不可变
- **配置系统**：运行时加载的配置对象一旦应用就完全冻结
- **响应式数据**：框架内部确保开发者不会意外修改深层属性
- **递归数据模型**：树形结构、嵌套的 JSON schema、递归的 AST 节点

这就是 `DeepReadonly` 的用武之地。

## 概念与定义

### Readonly 的本质

TypeScript 的 `readonly` 修饰符是类型系统层面的约束，它只存在于**编译期**。它在两个层面起作用：

1. **赋值检查**：禁止对标记为 `readonly` 的属性进行写操作
2. **类型标记**：是 TypeScript 结构类型系统的一部分，影响类型兼容性

```typescript
interface Mutable {
  x: number;
}
interface Immutable {
  readonly x: number;
}

const a: Mutable = { x: 1 };
const b: Immutable = a; // OK — 可变的赋给只读的，安全
a.x = 2;                // OK — 原始引用仍是可变的
// b.x = 2;             // ❌ Cannot assign to 'x' because it is a read-only property
```

### DeepReadonly 的定义

`DeepReadonly<T>` 的目标是：**递归地将 T 的所有属性（以及所有嵌套对象的属性）标记为 `readonly`**。

```typescript
// 使用期望
type DeepReadonlyConfig = DeepReadonly<Config>;
// 期望：
// {
//   readonly server: {
//     readonly host: string;
//     readonly port: number;
//   };
//   readonly debug: boolean;
// }
```

## 最小示例

一个能正常运行的最简实现：

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};

// 验证
type Test = {
  a: number;
  b: {
    c: string;
    d: readonly number[];
  };
  e: () => void;
};

type DeepTest = DeepReadonly<Test>;
// {
//   readonly a: number;
//   readonly b: {
//     readonly c: string;
//     readonly d: readonly number[];
//   };
//   readonly e: () => void;
// }
```

## 核心知识点拆解

### 1. 映射类型 + readonly 修饰符组合

`DeepReadonly` 的核心骨架是一个**递归的映射类型**：

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: 下一步_处理_T[K];
};
```

关键点解读：

- `readonly [K in keyof T]`：对 T 的每个属性 K 添加 readonly 修饰符
- `T[K]`：获取属性 K 的值类型
- 值类型如果是对象 → 递归调用 `DeepReadonly`
- 值类型如果是原始类型 → 直接返回

### 2. 边界条件：函数类型

函数（`() => void`）在 TypeScript 中也是 `object`：

```typescript
type IsObject<T> = T extends object ? true : false;
type Test = IsObject<() => void>; // true — 函数也是 object
```

所以不加边界检查的话：

```typescript
type DeepReadonlyBad<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonlyBad<T[K]>
    : T[K];
};

type BadResult = DeepReadonlyBad<{
  fn: () => void;
}>;
// 结果：{ readonly fn: DeepReadonlyBad<() => void> }
// DeepReadonly 应用到了函数类型上：
// 函数类型的 key 包括 call、bind、apply 等
// 结果变成了 { readonly call: ...; readonly bind: ...; readonly apply: ... }
// 完全不是我们想要的！
```

正确的处理方式是**将函数类型作为基本情况排除**：

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K]  // 函数类型直接保留
    : T[K] extends object
      ? DeepReadonly<T[K]>
      : T[K];
};
```

### 3. 边界条件：数组与元组

数组和元组也是 `object`，但处理方式需要特别考虑：

```typescript
// 数组的处理
type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends (...args: any[]) => any
    ? T
    : T extends object
      ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
      : T;

// 或者更精确地区分元组和数组：
type DeepReadonly<T> = T extends readonly any[]
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T extends (...args: any[]) => any
    ? T
    : T extends object
      ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
      : T;
```

但这里有个微妙的点：`T extends any[]` 的检查必须在 `T extends object` 之前，否则数组会被对象分支捕获。

### 4. 深度可选：DeepPartial 的对称实现

理解了 `DeepReadonly` 后，`DeepPartial` 的实现几乎是对称的：

```typescript
type DeepPartial<T> = T extends (infer U)[]
  ? DeepPartial<U>[]
  : T extends (...args: any[]) => any
    ? T
    : T extends object
      ? { [K in keyof T]?: DeepPartial<T[K]> }
      : T;

// 对比
// DeepReadonly:  readonly [K in keyof T]   + 递归
// DeepPartial:   [K in keyof T]?            + 递归
// 差异仅在 ? 和 readonly 修饰符的位置
```

### 5. 进阶：DeepRequired（深度去除可选）

```typescript
type DeepRequired<T> = T extends (infer U)[]
  ? DeepRequired<U>[]
  : T extends (...args: any[]) => any
    ? T
    : T extends object
      ? { [K in keyof T]-?: DeepRequired<T[K]> }
      : T;

// 使用 -? 去除可选标记
```

### 6. 组合拳：DeepFreeze

实际项目中，你可能需要同时做到"深度只读"和"深度不可变"：

```typescript
// 一个更完备的不可变类型
type DeepImmutable<T> = {
  readonly [K in keyof T]: T[K] extends Function
    ? T[K]
    : T[K] extends Map<infer K, infer V>
      ? ReadonlyMap<DeepImmutable<K>, DeepImmutable<V>>
      : T[K] extends Set<infer V>
        ? ReadonlySet<DeepImmutable<V>>
        : T[K] extends (infer U)[]
          ? ReadonlyArray<DeepImmutable<U>>
          : T[K] extends object
            ? DeepImmutable<T[K]>
            : T[K];
};
```

## 实战案例

### 案例1：全局配置系统的不可变性

在一个通用前端框架中，配置一旦生效就不应被修改：

```typescript
interface FrameworkConfig {
  server: {
    host: string;
    port: number;
    ssl: {
      enable: boolean;
      cert: string;
      key: string;
    };
    middlewares: Array<{
      name: string;
      options: Record<string, unknown>;
    }>;
  };
  rendering: {
    ssr: boolean;
    template: {
      path: string;
      engine: 'ejs' | 'pug' | 'handlebars';
      cache: boolean;
    };
  };
  database: {
    type: 'mysql' | 'postgres' | 'sqlite';
    connection: {
      host: string;
      port: number;
      username: string;
      password: string;
      pool: {
        min: number;
        max: number;
        timeout: number;
      };
    };
  };
}

// 冻结整个配置
type FrozenConfig = DeepReadonly<FrameworkConfig>;

// 工厂函数
function createConfig(overrides?: DeepPartial<FrozenConfig>): FrozenConfig {
  return Object.freeze(merge(defaults, overrides ?? {}));
}

const config = createConfig({
  server: {
    port: 3000,  // 部分覆盖
  },
});

// 下面这些操作都会报类型错误：
// config.server.port = 8080;         ❌ readonly
// config.server.ssl.enable = false;   ❌ 深层 readonly
// config.database.connection.pool.min = 1; ❌ 深层 readonly

// 但是可以安全读取任何值：
const port = config.server.port;                    // ✅ OK
const engine = config.rendering.template.engine;    // ✅ OK
```

### 案例2：状态管理中的不可变状态树

在实现一个类 Redux 的状态管理库时：

```typescript
type Reducer<S, A> = (state: DeepReadonly<S>, action: A) => S;

// 整个 state 树在 reducer 中完全是只读的
interface AppState {
  user: {
    profile: {
      name: string;
      email: string;
      preferences: {
        theme: 'light' | 'dark';
        language: string;
        notifications: boolean;
      };
    };
    posts: Array<{
      id: string;
      title: string;
      content: string;
      tags: string[];
    }>;
  };
  ui: {
    sidebar: {
      collapsed: boolean;
      width: number;
    };
    modals: string[];
  };
}

const rootReducer: Reducer<AppState, { type: string; payload: any }> = (
  state,
  action
) => {
  // state 中的每个嵌套属性都是 readonly
  // state.user.profile.name = 'new'; // ❌ 类型错误！
  // state.user.profile.preferences.theme = 'dark'; // ❌ 类型错误！

  switch (action.type) {
    case 'UPDATE_PROFILE':
      return {
        ...state,
        user: {
          ...state.user,
          profile: {
            ...state.user.profile,
            ...action.payload,
          },
        },
      };
    default:
      return state;
  }
};
```

### 案例3：表单校验库的字段状态标记

在一个动态表单库中，确保已校验的字段不可修改：

```typescript
interface FieldState {
  value: unknown;
  touched: boolean;
  dirty: boolean;
  errors: string[];
  valid: boolean;
}

interface FormState {
  fields: Record<string, FieldState>;
  valid: boolean;
  submitting: boolean;
}

// 提交后的表单状态不可修改
type SubmittedFormState = DeepReadonly<FormState>;

// 但在填写过程中允许修改脏状态的字段
type EditingFormState = FormState & {
  fields: {
    [key: string]: DeepPartial<FieldState> & { dirty: true };
  };
};

// 更实际的使用方式：表单校验完成后冻结
function submitForm(
  state: FormState
): SubmittedFormState | { errors: string[] } {
  const errors = validateForm(state);
  if (errors.length > 0) return { errors };

  // 冻结整个状态并提交
  return Object.freeze({
    ...state,
    submitting: true,
  }) as SubmittedFormState;
}
```

### 案例4：结合 ReadonlyMap 和 ReadonlySet

在需要处理 ES6 集合类型时：

```typescript
type DeepReadonlyAdvanced<T> = T extends Map<infer K, infer V>
  ? ReadonlyMap<DeepReadonlyAdvanced<K>, DeepReadonlyAdvanced<V>>
  : T extends Set<infer V>
    ? ReadonlySet<DeepReadonlyAdvanced<V>>
    : T extends WeakMap<infer K extends object, infer V>
      ? WeakMap<DeepReadonlyAdvanced<K>, DeepReadonlyAdvanced<V>>
      : T extends WeakSet<infer V extends object>
        ? WeakSet<DeepReadonlyAdvanced<V>>
        : T extends (...args: any[]) => any
          ? T
          : T extends object
            ? { readonly [K in keyof T]: DeepReadonlyAdvanced<T[K]> }
            : T;

// 验证
interface DataWithMap {
  name: string;
  metadata: Map<string, {
    createdAt: Date;
    updatedAt: Date;
    tags: Set<string>;
  }>;
}

type FrozenData = DeepReadonlyAdvanced<DataWithMap>;
// metadata 成为 ReadonlyMap
// 内部的 tags 成为 ReadonlySet
// 所有嵌套对象都变为 readonly
```

## 底层原理

### 映射类型中的修饰符处理

TypeScript 编译器在处理映射类型时，对修饰符遵循特定的规则：

```typescript
// 映射类型的修饰符语法
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];  // 去除 readonly
};

type Partial<T> = {
  [K in keyof T]?: T[K];          // 添加可选
};

type Required<T> = {
  [K in keyof T]-?: T[K];          // 去除可选
};

// 可以组合使用
type WritableRequired<T> = {
  -readonly [K in keyof T]-?: T[K]; // 去除 readonly 并去除可选
};
```

TypeScript 编译器内部是这样处理修饰符的：

1. 对原始类型中的每个属性，检查是否有 `readonly` 或 `?` 标记
2. 如果在映射类型中使用了 `readonly`（无 `-`），则在结果中**添加** readonly 修饰符
3. 如果在映射类型中使用了 `-readonly`，则在结果中**移除** readonly 修饰符
4. 如果在映射类型中使用了 `?`（无 `-`），则在结果中**添加**可选标记
5. 如果在映射类型中使用了 `-?`，则在结果中**移除**可选标记

### 递归类型的评估策略

TypeScript 编译器采用**惰性求值（Lazy Evaluation）**策略处理递归类型：

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};

// 编译器不会立即展开所有递归层级
// 只在使用处按需展开
// 这既限制了递归深度（默认 ~50 层），也保证了性能
```

TypeScript 编译器对递归类型有**递归深度限制**（通常约为 50 层，不同版本有所差异），超出会报错：

```typescript
// 极深嵌套可能导致 Type instantiation is excessively deep and possibly infinite.
type DeepNested = DeepReadonly<Nested50LevelsDeep>; // 可能超出限制
```

### 与 Object.freeze 的关系

`DeepReadonly` 只是编译期类型约束，而 `Object.freeze` 是运行时的真正冻结：

```typescript
function deepFreeze<T>(obj: T): DeepReadonly<T> {
  if (obj === null || typeof obj !== 'object') return obj as any;

  // 先冻结自身
  Object.freeze(obj);

  // 递归冻结所有可枚举属性
  for (const key of Object.keys(obj)) {
    const value = (obj as any)[key];
    if (typeof value === 'object' && value !== null && !Object.isFrozen(value)) {
      deepFreeze(value);
    }
  }

  return obj as DeepReadonly<T>;
}

// 使用：编译期 + 运行时双重保障
const config = deepFreeze(loadConfig());
// 编译期：DeepReadonly 类型检查
// 运行时：Object.freeze 保护
```

## 高频面试题解析

### Q1: 请手写实现 `DeepReadonly<T>`，并说明需要处理哪些边界情况

**解析**：这是 type-challenges 中等难度中的标准题目，考察递归类型和边界处理能力。

```typescript
// 完整实现
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends Record<string, unknown>
    ? T[K] extends Function
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};

// 边界情况：
// 1. 函数类型 —— 函数也是 object，需要特别处理，否则会深度限制报错
// 2. 原始类型（string, number, boolean, symbol, undefined, null）
// 3. 数组和元组
// 4. Map / Set 等内置对象
// 5. Date / RegExp 等特殊对象
```

### Q2: `DeepReadonly` 和 `Readonly` 有什么区别？为什么需要深度版本？

**解析**：考察对浅层 vs 深层不可变性的理解。

```typescript
interface Deep {
  readonly outer: {
    inner: number;
  };
}

// Readonly 是浅层的
type Shallow = Readonly<Deep>;
// readonly outer: { inner: number }
// 可以修改：deep.outer.inner = 5 ✅（类型不阻止）

// DeepReadonly 是深层的
type FullReadonly = DeepReadonly<Deep>;
// readonly outer: { readonly inner: number }
// 不允许：full.outer.inner = 5 ❌
```

**实际价值**：在不可变数据模式（Immutable Data Pattern）中，浅层只读只能防范第一层赋值，无法保证深层数据的不可变性。当你有 5 层嵌套的对象时，`Readonly` 的防范力只有 20%，而 `DeepReadonly` 是 100%。

### Q3: 如何处理 `DeepReadonly` 中数组/元组类型？

**解析**：考察具体实现细节。

```typescript
// 方案一：将数组映射为 readonly 元组
type DeepReadonlyV1<T> = T extends (infer U)[]
  ? readonly DeepReadonlyV1<U>[]
  : T extends Function
    ? T
    : T extends object
      ? { readonly [K in keyof T]: DeepReadonlyV1<T[K]> }
      : T;

// 方案二：直接使用映射类型（利用 keyof 对数组的增强支持）
type DeepReadonlyV2<T> = {
  readonly [K in keyof T]: T[K] extends Function
    ? T[K]
    : T[K] extends object
      ? DeepReadonlyV2<T[K]>
      : T[K];
};

// 区别：
// V1 将 [string, number] 变为 readonly [string, number]
// V2 将 [string, number] 变为 { readonly 0: string; readonly 1: number; length: ... }
// 推荐方案：先判断数组
```

### Q4: 实现 `DeepMutable<T>`（即 `DeepReadonly` 的反操作）

**解析**：考察对 `-readonly` 修饰符的掌握。

```typescript
type DeepMutable<T> = T extends readonly any[]
  ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
  : T extends Function
    ? T
    : T extends object
      ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
      : T;

type ImmutableConfig = Readonly<{
  server: Readonly<{
    host: string;
    port: number;
  }>;
}>;

type MutableConfig = DeepMutable<ImmutableConfig>;
// 所有 readonly 都被递归去除

// 验证
const config: MutableConfig = { server: { host: 'localhost', port: 3000 } };
config.server.port = 8080; // ✅ OK
config.server.host = 'dev.local'; // ✅ OK
```

### Q5: DeepReadonly 在 Redux/Vuex 等状态管理库中有什么实际应用？

**解析**：考察理论落地到框架实践的能力。

```typescript
// 在 redux/toolkit 风格中
type AppAction = {
  type: string;
  payload?: unknown;
};

// Reducer 中的 state 应该是不可变的
type Reducer<S> = (
  state: DeepReadonly<S>,
  action: AppAction
) => S;

// Vue 3 的 reactive 响应式数据
// 虽然不是直接使用 DeepReadonly，但 Vue 3 内部的
// 类型定义使用了类似的递归类型模式

// Vue 3 Script Setup 中的 defineProps
// interface Props { items: { id: number; label: string }[] }
// const props = defineProps<Props>();
// props.items 中的每个元素都可以直接读取但只读

// React useReducer 也是一样的模式
function useTypedReducer<S, A>(
  reducer: (state: DeepReadonly<S>, action: A) => S,
  initialState: S
) {
  return useReducer(
    (state: S, action: A) => reducer(state as DeepReadonly<S>, action),
    initialState
  );
}
```

## 总结与扩展

### 核心要点回顾

1. **`DeepReadonly` 的本质**：递归的映射类型 + 正确的边界条件
2. **边界处理**：函数类型、数组/元组、Map/Set 等特殊对象必须单独处理
3. **对称模式**：`DeepReadonly` / `DeepMutable` / `DeepPartial` / `DeepRequired` 是四件套，它们共享相同的递归结构，差异仅在修饰符
4. **运行时配合**：`Object.freeze()` + `DeepReadonly` 提供编译期+运行时双重保障

### 进阶方向

1. **更精细的控制**：通过泛型参数控制递归深度（如 `DeepReadonly<T, Depth extends number = 3>`）
2. **有选择的深度只读**：某些路径跳过深度处理
3. **与其他工具类型组合**：`DeepReadonly<Required<T>>` 组合使用
4. **非递归替代方案**：对于已知固定深度的类型，可以用非递归方案提高性能
5. **了解映射类型的性能开销**：深度嵌套的类型递归在大型项目中可能影响编辑器性能

通过掌握 DeepReadonly，你已经迈入了 TypeScript 高级类型编程的大门。这个模式（递归映射类型 + 边界处理）会反复出现在各种工具类型实现中。如果你想挑战更多，可以去 type-challenges 上搜索 "readonly" 关键词，尝试实现 `DeepReadonly` 的不同变体。
