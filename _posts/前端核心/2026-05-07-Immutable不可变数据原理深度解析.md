---
title: "Immutable不可变数据原理深度解析"
date: 2026-05-07
categories: ["前端核心", "JavaScript"]
tags: ["JavaScript", "不可变数据", "Immer", "函数式编程", "性能优化"]
description: "深入解析不可变数据的概念、结构共享机制、Immer.js原理及React/Vue中的应用"
---

## 一句话概括

不可变数据是一种**创建数据后永不修改，变更时返回新引用**的设计模式，通过结构共享实现高效内存使用，是React/Vue性能优化的核心基础。

## 背景

在现代前端开发中，不可变数据是一个高频出现却又容易被忽视的概念：

- **React的状态更新**：为什么不能直接修改state？为什么要用`setState(newState)`？
- **Vue的响应式系统**：为什么Vue3用Proxy替代了Object.defineProperty？
- **Redux的设计哲学**：为什么reducer必须是纯函数，不能修改原状态？

这些问题的答案都指向同一个概念：**不可变数据（Immutable Data）**。

面试中，不可变数据是框架原理考察的必考点：
- "为什么React中不能直接修改state？"
- "Immer.js是怎么实现高效不可变更新的？"
- "什么是结构共享？它解决了什么问题？"

理解不可变数据，才能真正理解现代前端框架的性能优化原理。

## 概念与定义

### 什么是不可变数据？

**不可变数据（Immutable Data）**：一旦创建，就不能被修改的数据。任何"修改"操作都会返回一个全新的数据结构，原数据保持不变。

```javascript
// 可变数据（Mutable）：直接修改原数据
const mutable = { name: 'Alice', age: 25 };
mutable.age = 26;  // 直接修改，原数据变了

// 不可变数据（Immutable）：返回新数据
const original = { name: 'Alice', age: 25 };
const updated = { ...original, age: 26 };  // 创建新对象
// original 仍然是 { name: 'Alice', age: 25 }
```

### 为什么需要不可变数据？

#### 1. **引用相等判断**

React的`shouldComponentUpdate`、`React.memo`、`useMemo`都依赖引用相等判断：

```javascript
// ❌ 直接修改：引用不变，React检测不到变化
const [user, setUser] = useState({ name: 'Alice' });
user.name = 'Bob';
setUser(user);  // 同一个引用，组件不会重新渲染

// ✅ 不可变更新：新引用，React能检测到变化
setUser({ ...user, name: 'Bob' });  // 新引用，触发重新渲染
```

#### 2. **时间旅行调试**

Redux DevTools的时间旅行功能依赖状态快照，如果直接修改原状态，就无法回溯。

#### 3. **并发安全**

多线程/多任务环境下，不可变数据不需要锁，天然线程安全。

### 不可变数据的代价

```javascript
// 深层嵌套对象的更新非常繁琐
const state = {
  user: {
    profile: {
      name: 'Alice',
      address: {
        city: 'Beijing'
      }
    }
  }
};

// 嵌套展开...噩梦
const newState = {
  ...state,
  user: {
    ...state.user,
    profile: {
      ...state.user.profile,
      address: {
        ...state.user.profile.address,
        city: 'Shanghai'
      }
    }
  }
};
```

这就是**Immer.js**要解决的问题。

## 最小示例

### 手动不可变更新

```javascript
// 数组不可变操作
const arr = [1, 2, 3];
const newArr = [...arr, 4];           // 添加
const filtered = arr.filter(x => x !== 2);  // 删除
const mapped = arr.map(x => x * 2);   // 修改

// 对象不可变操作
const obj = { a: 1, b: 2 };
const newObj = { ...obj, c: 3 };      // 添加属性
const { a, ...rest } = obj;            // 删除属性（ES2020）
const updated = { ...obj, a: 10 };    // 修改属性
```

### 使用Immer简化

```javascript
import { produce } from 'immer';

const state = {
  user: {
    profile: {
      name: 'Alice',
      address: { city: 'Beijing' }
    }
  }
};

// ✅ 使用Immer：写法像直接修改，实际是不可变更新
const newState = produce(state, draft => {
  draft.user.profile.address.city = 'Shanghai';
});

// 原状态保持不变
console.log(state.user.profile.address.city);  // 'Beijing'
console.log(newState.user.profile.address.city);  // 'Shanghai'
```

## 核心知识点拆解

### 1. 不可变数据的实现方式

#### 方式一：原生JavaScript（浅拷贝）

```javascript
// Object.assign（第一层是浅拷贝）
const original = { a: 1, b: { c: 2 } };
const copied = Object.assign({}, original, { a: 10 });
// copied.b 和 original.b 指向同一个对象！

// 展开运算符（同上）
const spread = { ...original, a: 10 };
// spread.b === original.b  → true（浅拷贝）

// 数组方法（返回新数组）
const arr = [1, 2, 3];
arr.push(4);       // ❌ 原地修改
arr.concat(4);     // ✅ 返回新数组
[...arr, 4];       // ✅ 展开运算符
```

#### 方式二：深拷贝（JSON + 递归）

```javascript
// JSON方法（有局限）
const deepCopy = JSON.parse(JSON.stringify(obj));
// 问题：无法处理函数、Symbol、循环引用、Date、RegExp等

// 递归深拷贝（支持更多类型）
function deepClone(obj, cache = new Map()) {
  if (obj === null || typeof obj !== 'object') return obj;
  if (cache.has(obj)) return cache.get(obj);
  
  const clone = Array.isArray(obj) ? [] : {};
  cache.set(obj, clone);
  
  for (const key in obj) {
    if (obj.hasOwnProperty(key)) {
      clone[key] = deepClone(obj[key], cache);
    }
  }
  return clone;
}
```

**问题**：每次更新都深拷贝整个对象，性能太差。

#### 方式三：结构共享（Persistent Data Structure）

这是**Immer.js**和**Immutable.js**的核心技术。

### 2. 结构共享机制（Structural Sharing）

**核心思想**：只复制"变化路径"上的节点，共享未变化的部分。

```javascript
const original = {
  a: { x: 1, y: 2 },
  b: { x: 3, y: 4 },
  c: 5
};

// 修改 a.x = 10
const updated = {
  a: { x: 10, y: 2 },  // 新的 a 对象
  b: original.b,      // 复用原 b（共享）
  c: original.c       // 复用原 c（共享）
};

// 内存对比
original.a === updated.a;  // false（a被复制）
original.b === updated.b;  // true（b被共享）
original.c === updated.c;  // true（c被共享）
```

**可视化结构共享**：

```
原始数据结构：
     root
    / | \
   a  b  c
  /|  /|  \
 x y x y  5
 1 2 3 4

修改 a.x = 10 后：
     root' (新)
    / | \
   a' b  c    ← b和c被共享（同一引用）
  /|
 x y
10 2
```

**性能优势**：
- 空间复杂度：O(变化路径深度)，而非O(总数据量)
- 时间复杂度：只复制必要节点，效率高

### 3. Immer.js原理深度解析

Immer是"不可变数据"与"便捷写法"的完美结合。

#### 核心概念：Draft（草稿）

```javascript
import { produce } from 'immer';

const state = { count: 0, list: [1, 2, 3] };

const nextState = produce(state, draft => {
  // draft 是 state 的"代理"
  // 对 draft 的修改会被记录，最终生成新状态
  draft.count = 1;
  draft.list.push(4);
});

// state 保持不变（不可变）
console.log(state);      // { count: 0, list: [1, 2, 3] }
console.log(nextState);  // { count: 1, list: [1, 2, 3, 4] }

// 结构共享：未变化的部分共享引用
state.list !== nextState.list;  // true（list被修改）
```

#### 实现原理：Proxy代理

```javascript
// 简化版Immer实现
function produce(base, recipe) {
  const changed = new Map();  // 记录修改
  const proxies = new Map();  // 缓存代理
  
  // 创建代理
  function createProxy(target) {
    if (proxies.has(target)) return proxies.get(target);
    
    const proxy = new Proxy(target, {
      get(obj, key) {
        // 返回嵌套对象的代理
        const value = obj[key];
        if (typeof value === 'object' && value !== null) {
          return createProxy(value);
        }
        return value;
      },
      
      set(obj, key, value) {
        changed.set(obj, true);  // 标记为已修改
        obj[key] = value;
        return true;
      }
    });
    
    proxies.set(target, proxy);
    return proxy;
  }
  
  // 执行修改
  const draft = createProxy(base);
  recipe(draft);
  
  // 生成新状态（结构共享）
  function finalize(obj) {
    if (!changed.has(obj)) return obj;  // 未修改，返回原引用
    
    const clone = Array.isArray(obj) ? [...obj] : { ...obj };
    for (const key in clone) {
      if (typeof clone[key] === 'object') {
        clone[key] = finalize(clone[key]);  // 递归处理
      }
    }
    return clone;
  }
  
  return finalize(base);
}
```

#### Immer的核心流程

```
1. 创建Draft：Proxy包装原始状态
   state → draft（代理对象）

2. 执行Recipe：在draft上"直接修改"
   draft.count = 1;  // 实际是记录修改

3. Finalize：生成新状态
   - 检测哪些节点被修改
   - 只复制修改路径上的节点
   - 共享未修改的节点
```

#### 自动冻结（Object.freeze）

```javascript
import { produce, enableFreeze } from 'immer';

// Immer默认在生产环境自动冻结结果
enableFreeze();  // 显式启用

const state = { a: 1 };
const newState = produce(state, draft => {
  draft.a = 2;
});

newState.a = 3;  // 报错！Cannot assign to read only property
```

### 4. React中的不可变数据

#### 为什么React需要不可变数据？

```jsx
function Counter() {
  const [state, setState] = useState({ count: 0, list: [] });
  
  // ❌ 错误：直接修改state
  const badIncrement = () => {
    state.count += 1;
    setState(state);  // 引用未变，不会触发渲染
  };
  
  // ✅ 正确：创建新对象
  const goodIncrement = () => {
    setState(prev => ({ ...prev, count: prev.count + 1 }));
  };
  
  // ✅ 使用Immer
  const immerIncrement = () => {
    setState(produce(state, draft => {
      draft.count += 1;
    }));
  };
}
```

#### React.memo与useMemo依赖引用

```jsx
// React.memo：只有props引用变化才重新渲染
const ExpensiveComponent = React.memo(({ data }) => {
  return <div>{data.name}</div>;
});

// ❌ 直接修改props.data，不会触发重渲染
data.name = 'new';

// ✅ 创建新对象，触发重渲染
const newData = { ...data, name: 'new' };

// useMemo：依赖项引用变化才重新计算
const result = useMemo(() => {
  return heavyCompute(data);
}, [data]);  // data引用变化才重新计算
```

### 5. Vue中的不可变数据需求

#### Vue 2：Object.defineProperty的局限

```javascript
// Vue 2响应式：无法检测新增属性
const vm = new Vue({
  data: { user: { name: 'Alice' } }
});

vm.user.name = 'Bob';  // ✅ 可检测
vm.user.age = 25;      // ❌ 无法检测新增属性
Vue.set(vm.user, 'age', 25);  // ✅ 需要手动调用
```

#### Vue 3：Proxy响应式

```javascript
import { reactive } from 'vue';

const state = reactive({
  user: { name: 'Alice' }
});

// Proxy可以检测新增属性
state.user.age = 25;  // ✅ 自动检测

// 但数组直接修改索引仍然有问题
const arr = reactive([1, 2, 3]);
arr[0] = 10;  // ✅ Vue 3能检测
arr.length = 0;  // ✅ Vue 3能检测
```

#### Vue的不可变数据最佳实践

```javascript
// ❌ 直接修改（虽然Vue 3能检测，但不推荐）
state.user.name = 'Bob';

// ✅ 创建新对象（推荐）
state.user = { ...state.user, name: 'Bob' };

// 使用Vue的reactive + 不可变更新
const state = reactive({ list: [] });
state.list = [...state.list, newItem];  // 不可变更新
```

## 实战案例

### 案例1：实现一个简易的状态管理（Redux模式）

```javascript
function createStore(reducer, initialState) {
  let state = initialState;
  const listeners = [];
  
  function getState() {
    return state;
  }
  
  function dispatch(action) {
    // reducer必须返回新状态（不可变）
    state = reducer(state, action);
    listeners.forEach(fn => fn());
  }
  
  function subscribe(listener) {
    listeners.push(listener);
    return () => {
      const index = listeners.indexOf(listener);
      listeners.splice(index, 1);
    };
  }
  
  return { getState, dispatch, subscribe };
}

// Reducer：必须是纯函数，返回新状态
const counterReducer = (state = { count: 0 }, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };  // 不可变更新
    case 'DECREMENT':
      return { ...state, count: state.count - 1 };
    default:
      return state;
  }
};

const store = createStore(counterReducer, { count: 0 });
```

### 案例2：复杂嵌套状态更新

```javascript
import { produce } from 'immer';

const initialState = {
  users: {
    byId: {
      '1': { id: '1', name: 'Alice', posts: [1, 2] },
      '2': { id: '2', name: 'Bob', posts: [3] }
    },
    allIds: ['1', '2']
  },
  posts: {
    byId: {
      1: { id: 1, title: 'Hello', likes: 10 },
      2: { id: 2, title: 'World', likes: 5 },
      3: { id: 3, title: 'Test', likes: 0 }
    },
    allIds: [1, 2, 3]
  }
};

// 给用户1的帖子1点赞
const newState = produce(initialState, draft => {
  draft.posts.byId[1].likes += 1;
  draft.users.byId['1'].postCount = draft.users.byId['1'].posts.length;
});

// 未变化的部分共享引用
initialState.posts.byId[2] === newState.posts.byId[2];  // true
initialState.users.byId['2'] === newState.users.byId['2'];  // true
```

### 案例3：列表性能优化

```jsx
import { produce } from 'immer';

function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Learn React', done: false },
    { id: 2, text: 'Learn Immer', done: false }
  ]);
  
  // ✅ 使用Immer切换完成状态
  const toggleTodo = (id) => {
    setTodos(produce(todos, draft => {
      const todo = draft.find(t => t.id === id);
      if (todo) todo.done = !todo.done;
    }));
  };
  
  // ❌ 不使用Immer的繁琐写法
  const toggleTodoMutable = (id) => {
    setTodos(todos.map(todo => 
      todo.id === id 
        ? { ...todo, done: !todo.done } 
        : todo
    ));
  };
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id} onClick={() => toggleTodo(todo.id)}>
          {todo.text} - {todo.done ? '✓' : '○'}
        </li>
      ))}
    </ul>
  );
}
```

## 底层原理

### Immer vs Immutable.js

| 特性 | Immer | Immutable.js |
|-----|-------|--------------|
| API风格 | 原生JS对象 | 自定义数据结构 |
| 学习成本 | 低 | 高（需学习新API） |
| 性能 | 中等（Proxy开销） | 高（专门优化的数据结构） |
| 结构共享 | 自动 | 自动 |
| 序列化 | 原生JSON | 需转换 |
| 包体积 | 16KB | 56KB |

### 结构共享的Trie树实现

Immutable.js使用**哈希数组映射前缀树（HAMT）**实现高效的结构共享：

```javascript
// 简化版HAMT概念
class TrieNode {
  constructor() {
    this.children = {};  // 子节点
    this.value = null;  // 叶节点存储值
  }
}

// 插入路径：a.b.c = value
function insert(root, path, value) {
  let node = root;
  for (const key of path) {
    if (!node.children[key]) {
      node.children[key] = new TrieNode();
    }
    node = node.children[key];
  }
  node.value = value;
}

// 路径压缩：共享前缀
// { a: { b: 1 } } 和 { a: { b: 2, c: 3 } }
// 共享 a 节点
```

### Copy-on-Write（写时复制）

Immer的核心思想是**Copy-on-Write**：

```javascript
// 只有修改时才复制
function getOrCreateChild(parent, key) {
  if (parent[key] && !isModified(parent[key])) {
    return parent[key];  // 共享
  }
  const child = shallowCopy(parent[key]);  // 复制
  parent[key] = child;
  return child;
}
```

## 高频面试题解析

### Q1：为什么React不能直接修改state？

**答案**：
1. **引用相等判断**：React使用`Object.is(prev, next)`判断状态是否变化，直接修改不改变引用，不会触发渲染
2. **shouldComponentUpdate优化**：React.memo、PureComponent依赖浅比较，直接修改会绕过优化
3. **并发模式安全**：React 18的并发渲染需要状态快照，直接修改会导致竞态条件

### Q2：Immer是怎么实现结构共享的？

**答案**：
1. 使用Proxy代理原始对象，拦截所有读写操作
2. 记录修改路径，标记哪些节点被修改
3. 生成新状态时，只复制修改路径上的节点，共享未修改的节点
4. 最终结果自动Object.freeze冻结（生产环境）

### Q3：什么场景应该用Immer？

**答案**：
适用场景：
- 深层嵌套状态更新（如Redux reducer）
- 复杂表单状态管理
- 不想写繁琐的展开运算符代码

不适用场景：
- 极致性能要求（Proxy有开销）
- 状态非常简单（浅层对象）
- 需要兼容不支持Proxy的环境

### Q4：Object.assign和展开运算符有什么区别？

**答案**：
```javascript
// 基本等价：都是浅拷贝
const a = Object.assign({}, obj, { key: value });
const b = { ...obj, key: value };

// 区别1：Object.assign会触发setter
const obj = {
  set foo(value) { console.log('setter called'); }
};
Object.assign(obj, { foo: 1 });  // 触发setter
{ ...obj, foo: 1 };              // 不触发

// 区别2：展开运算符不能展开undefined/null
{ ...null };     // {}
Object.assign({}, null);  // 不报错
```

### Q5：手写一个简单的结构共享更新函数

```javascript
function updateImmutable(obj, path, value) {
  if (path.length === 0) return value;
  
  const [key, ...rest] = path;
  const newObj = Array.isArray(obj) ? [...obj] : { ...obj };
  
  newObj[key] = updateImmutable(obj[key] || {}, rest, value);
  return newObj;
}

// 使用
const state = { a: { b: { c: 1 } } };
const newState = updateImmutable(state, ['a', 'b', 'c'], 2);
// state.a.b.c = 1, newState.a.b.c = 2
// state.a === newState.a  → false（a.b路径上的节点都被复制）
```

## 总结与扩展

### 核心要点回顾

1. **不可变数据**：创建后永不修改，变更返回新引用
2. **必要性**：引用相等判断、时间旅行调试、并发安全
3. **结构共享**：只复制修改路径，共享未变部分，实现高效不可变更新
4. **Immer.js**：Proxy + Copy-on-Write，让不可变更新像直接修改一样简单
5. **框架应用**：React的状态更新、Redux的reducer、Vue 3的响应式都需要不可变数据

### 深入学习资源

- [Immer官方文档](https://immerjs.github.io/immer/)
- [Immutable.js官方文档](https://immutable-js.com/)
- [Morris的论文：Persistent Data Structures](https://www.cs.cmu.edu/~sleator/papers/another-persistent.pdf)
- [React官方文档：状态更新是只读的](https://react.dev/learn/updating-objects-in-state)

### 相关主题

- 深拷贝与浅拷贝的实现
- Redux中间件原理
- Vue 3响应式原理
- 函数式编程范式

---

> 不可变数据是现代前端框架的基石。理解它，才能真正理解React的渲染机制、Redux的设计哲学、以及Vue 3的响应式原理。掌握Immer，让复杂状态更新变得简单高效。
