---
layout: post
title: "类型挑战：DeepReadonly"
date: 2026-02-19 00:00:00 +0800
categories: ["前端核心", "TypeScript"]
tags: ["TypeScript", "类型体操", "DeepReadonly", "递归类型", "面试题"]
---

## 一句话概括

`DeepReadonly<T>` 把对象及其所有嵌套子对象的属性全部递归标记为 `readonly`。它是 type-challenges 的中等标杆题，考的不是语法，而是**分支顺序**：数组 → 函数 → 对象 → 原始类型，一步走错全盘变形。

## 核心知识点

### 1. 第一版——看起来很对，实际埋了大雷

```ts
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>
    : T[K];
};

// 问题在哪？
type Fn = () => void;
// DeepReadonly<Fn> → 会把 call、apply、bind、arguments 全变成 readonly
// 函数彻底变形了！
```

### 2. 函数拦截——必须在递归前拦住

```ts
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K]                           // 函数直接原样保留
    : T[K] extends object
      ? DeepReadonly<T[K]>
      : T[K];
};
```

**为什么 `extends (...args: any[]) => any` 能捕获所有函数？** 因为函数在 TS 里是 `object` 的子类型，`(() => {}) extends object` 是 `true`。不拦截它，它就会掉进对象递归分支。

### 3. 数组也必须单独处理

```ts
// 如果用映射类型遍历数组：
// keyof number[] → number | 'length' | 'push' | 'pop' | ...
// 结果变成 { readonly 0: ..., readonly length: ..., readonly push: ... }
// 这根本不是数组了！

// ✅ 正确：数组用 ReadonlyArray 递归元素
type DeepReadonly<T> =
  T extends (infer U)[]
    ? ReadonlyArray<DeepReadonly<U>>    // 数组排第一
    : T extends (...args: any[]) => any // 函数排第二
      ? T
      : T extends object               // 对象排第三
        ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
        : T;                            // 原始类型兜底
```

**分支顺序为什么必须这样？** `T extends any[]` 必须写在 `T extends object` 前面——因为 `any[]` 也是 `object`，先走到对象分支数组就变形了。

### 4. DeepPartial / DeepRequired——改一个修饰符

```ts
type DeepPartial<T> = T extends (infer U)[]
  ? DeepPartial<U>[]
  : T extends Function ? T
  : T extends object ? { [K in keyof T]?: DeepPartial<T[K]> }  // 就改了 ?
  : T;

type DeepRequired<T> = T extends (infer U)[]
  ? DeepRequired<U>[]
  : T extends Function ? T
  : T extends object ? { [K in keyof T]-?: DeepRequired<T[K]> } // -? 去可选
  : T;
```

理解 DeepReadonly 后，DeepPartial/DeepRequired 只是把 `readonly` 换成 `?` / `-?`。

### 5. 编译 + 运行时双保险——DeepFreeze

```ts
function deepFreeze<T>(obj: T): DeepReadonly<T> {
  Object.freeze(obj);
  for (const val of Object.values(obj)) {
    if (val && typeof val === 'object' && !Object.isFrozen(val)) {
      deepFreeze(val);
    }
  }
  return obj as any;
}

const config = deepFreeze({ server: { host: 'localhost', port: 3000 } });
// config.server.port = 8080; // ❌ 编译报错 + 运行时静默失败
```

## 其实你每天都在用

- **Vue 3 `defineProps`：** 传入组件的 props 编译后被 `DeepReadonly` 包裹，子组件 `props.name = 'x'` 直接类型报错
- **Redux / Zustand 的 state：** reducer 返回的新 state 要求只读，直接 mutate 被 TS 拦住
- **环境变量 `import.meta.env`：** 编译时就是只读类型，运行时也改不了
- **前端常量配置：** `const CONFIG = deepFreeze({ api: { baseUrl: '...' } })` 告别全局污染
- **React.memo 配合使用：** 只读类型让开发者不敢直接 mutate props，只能用不可变更新

## 常见误解（FAQ）

- **❌ 误区：「`Readonly` 够了」** `Readonly` 只浅层。五层嵌套对象只有第一层被保护，剩下 80% 的属性照改不误。嵌套超过一层就必须 `DeepReadonly`。

- **❌ 误区：「函数可以不管，跳过就行」** 必须拦截，不能靠"反正没人会去改函数的属性"的心理安慰。不拦截的话 TS 会展开函数的所有原型属性（call、apply、bind……），轻则无意义膨胀类型，重则递归超限报错。

- **❌ 误区：「数组不需要单独分支」** 映射类型遍历数组会把 `keyof []` 包含的 `length`、`push`、索引都变成 readonly 属性，结果是一个"畸形的对象"，不是数组类型。

- **❌ 误区：「DeepReadonly 拖慢类型检查」** 3-5 层配置对象几乎零性能影响。但递归 40+ 层的自引用类型（如 AST 节点带 `parent` 指针）会触 TS "excessively deep" 限制，需带深度参数 `Depth extends number = 5` 做截断。

## 一句话总结

DeepReadonly 的四个分支顺序是"面试判死刑题"：数组在最前 → 函数拦截 → 对象递归 → 原始类型兜底——顺序一乱，输出全废。
