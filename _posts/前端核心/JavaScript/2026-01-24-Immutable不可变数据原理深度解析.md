---
title: "Immutable不可变数据原理深度解析"
date: 2026-01-24
categories: ["前端核心", "JavaScript"]
tags: ["JavaScript", "不可变数据", "Immer", "函数式编程", "性能优化"]
description: "深入解析不可变数据的概念、结构共享机制、Immer.js原理及React/Vue中的应用"
---

## 一句话概括

不可变数据 = 数据一旦创建就不再原地修改，任何"改"都是产出一个新副本。配合**结构共享**只复制变化路径上的节点，让深嵌套更新也能既快又不会弄脏原数据。这就是 React 的 setState、Redux 的 reducer、Vue 的响应式能"一眼看出变了没"的底层基础。

## 核心知识点

### 1. 什么是不可变数据？一句话 + 一个例子就够了

不可变数据的本质：**永远不碰原对象，每次变更都返回一个新引用**。

```js
// ❌ 可变：原地改，引用不变
const user = { name: 'Alice', age: 25 };
user.age = 26;
console.log(user); // { name: 'Alice', age: 26 }，原对象已被污染

// ✅ 不可变：返回新对象，原对象纹丝不动
const updated = { ...user, age: 26 };
console.log(user.age);    // 25 — 原对象完好
console.log(updated.age); // 26
```

面试官要听的不是你背定义，而是你明白它和"引用比较"之间的关系。

### 2. 为什么需要不可变数据？—— 引用相等是命门

React 的 PureComponent、memo、useMemo、useEffect 的依赖数组，**全是靠 `===` 判断变了没**。不可变数据让 `===` 判断变得 O(1)：

```js
const prev = { user: { name: 'Alice' } };
const next = { ...prev, user: { ...prev.user, name: 'Bob' } };

prev === next;         // false ✅ 根变了
prev.user === next.user; // false ✅ user 变了
```

如果原地修改 `prev.user.name = 'Bob'`，`prev === next` 是 `true`，React 直接跳过渲染 — 这几乎是每个新手都会踩的坑。

### 3. 结构共享：只拷贝"变了的"，其余全复用

深拷贝整个对象树太浪费。结构共享的思路是：**沿修改路径逐层浅拷贝，路径以外的节点直接复用原引用**。

```js
const base = {
  a: { x: 1, y: 2 },
  b: { x: 3, y: 4 },
};

// "修改 a.x 为 10"
const next = {
  a: { ...base.a, x: 10 }, // a 是新对象
  b: base.b,               // b 直接复用
};

base.a === next.a; // false — 沿路径被拷贝
base.b === next.b; // true  — 没碰到，直接共享
```

结构共享把更新成本从 O(n) 降到 O(path depth)，这才是 Immer 和 Immutable.js 性能不掉链子的核心。

### 4. Immer 原理：Proxy + Copy-on-Write

Immer 让你"假装"直接修改，实际在 Proxy 的掩护下默默记录变更路径：

```js
import { produce } from 'immer';

const state = { user: { name: 'Alice', tags: ['js'] } };

const nextState = produce(state, draft => {
  draft.user.name = 'Bob';
  draft.user.tags.push('react');
});

// state 纹丝不动
console.log(state.user.name);      // 'Alice'
console.log(state.user.tags);      // ['js']
// 未触及的节点结构共享
console.log(state.user === nextState.user); // false（user 在修改路径上）
```

简化版核心思路：

```js
function produce(base, recipe) {
  const dirty = new WeakMap(); // 记录哪些对象被"碰过"

  const createProxy = target => new Proxy(target, {
    get(obj, key) {
      const val = obj[key];
      if (typeof val === 'object' && val !== null) return createProxy(val);
      return val;
    },
    set(obj, key, value) {
      dirty.set(obj, true);
      obj[key] = value;
      return true;
    },
  });

  const draft = createProxy(base);
  recipe(draft);

  // finalize: 脏的浅拷贝，干净的复用原引用
  const finalize = obj =>
    dirty.has(obj)
      ? Object.assign(Array.isArray(obj) ? [...obj] : { ...obj },
          ...Object.keys(obj).map(k => {
            const v = obj[k];
            return typeof v === 'object' && v !== null ? { [k]: finalize(v) } : { [k]: v };
          }))
      : obj;

  return finalize(base);
}
```

这段简化版省略了数组 push 等方法的代理，但 Proxy → 标记脏节点 → 沿路径拷贝这条链路一览无余。

### 5. React 中的不可变数据实践

```jsx
// ❌ 直接 mutate：引用未变，React 直接不渲染
setState(prev => { prev.count = 1; return prev; });

// ✅ 不可变：新引用触发渲染
setState(prev => ({ ...prev, count: prev.count + 1 }));

// ✅ Immer 语法糖：写着爽，跑起来也正确
import { useImmer } from 'use-immer';
const [state, updateState] = useImmer({ count: 0, list: [] });
updateState(draft => { draft.count += 1; draft.list.push(42); });
```

## 其实你每天都在用

**1. React 的 setState**
你每次 `setState({ ...prev, key })` 就是在做不可变更新。React 内部 `Object.is(prev, next)` 一比较，新引用 → 触发渲染。

**2. Array 的 map/filter/reduce**
这些方法都返回新数组，从不碰原数组。`arr.map(x => x * 2)` 就是不可变思想的日常实践。

**3. Redux / Zustand 的 reducer**
Redux 要求 reducer 是纯函数 — 输入 state + action，输出新 state，绝不能原地改。你不遵守的话时间旅行就直接崩了。

**4. Vue 的 computed**
`computed` 依赖的 ref/reactive 变了，computed 返回新值。虽然 Vue 的响应式允许直接改，但 Vuex/Pinia 的 store 设计本质也是在推崇不可变更新。

**5. Git 的 commit**
每次 commit 不修改历史 snapshot，而是创建一个新的。整个 Git 的 object store 就是结构共享的典范 — 没变的文件直接复用 blob hash。

## 常见误解

**❌ 误区 1："展开运算符 `{...obj}` 是深拷贝"**
它是浅拷贝。嵌套对象仍共享引用，改内层照样会污染原对象。真要深拷贝不丢引用关系，得用递归或 `structuredClone`。

**❌ 误区 2："Immer 内部做了深拷贝"**
Immer 恰恰不做深拷贝。它靠 Proxy 记录哪些节点被修改，只浅拷贝脏节点路径，其余全部共享原引用。深拷贝就走样了，性能反而崩。

**❌ 误区 3："Vue 3 的 Proxy 可以随意直接改，不需要不可变"**
Vue 3 确实能检测到直接修改，但 **不等于** 你应该随意 mutate。大规模应用里，不可变更新让数据流可预测、更好调试；想在 Vuex/Pinia 里拆 action 做时间旅行，必须走不可变路线。

**❌ 误区 4："`Object.freeze` 就是不可变数据"**
`Object.freeze` 只是浅冻结第一层，不让修改当前对象属性。它不负责深层冻结，也不帮你生成新版本 — 它是"锁住"，不是"不可变更新机制"。Immer 自动 freeze 结果是附加保护，不是不可变本身。

## 一句话总结

不可变数据的核心不是"不能改"，而是"改了就换" — 换一个引用的代价换来 O(1) 变更检测和可追溯的状态历史，这才是现代前端框架性能优化和调试体验的根。
