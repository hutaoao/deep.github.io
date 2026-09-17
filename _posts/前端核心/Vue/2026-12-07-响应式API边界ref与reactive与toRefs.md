---
layout: post
title: "响应式 API 的边界：ref / reactive / toRefs 什么时候会断"
date: 2026-12-07 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [Vue响应式, ref, reactive, toRefs, shallowRef]
description: >
  把 Vue 响应式的边界问题收敛成一条底层规则——追踪只发生在"通过 proxy 的读写"上；
  由此推出解构断链、重新赋值断链、传参断链三类失效场景的准确边界（含实测：解构出的嵌套对象
  仍是响应式代理）、ref 的完整解包规则（数组/Map/浅层响应式都不解包）、模板插值为什么"看起来正常"，
  以及 toRef / toRefs / shallowRef / triggerRef 的正确用法。
---

## 一句话概括

响应式 API 的面试题，九成的坑都源于同一句话：

> **Vue 只能追踪"通过 proxy 发生的读写"，它追踪不了你的变量绑定。**

`reactive()` 返回的是一个 Proxy，你写 `state.count` 时走的是 get 拦截器，Vue 在这里做依赖收集；写 `state.count = 1` 走 set 拦截器，Vue 在这里触发更新。**而 JS 变量本身只是内存地址的别名，它不在 Vue 的观测范围内。**

所以后面所有"什么时候会丢响应式"的问题，都不是需要背的零散知识点，而是这句话的推论：

- 解构 = 把属性值读出来存进新变量 → 新变量不认识 proxy → 断链；
- 重新赋值 `state = reactive({...})` = 变量指向了新代理，旧依赖还挂在旧代理上 → 断链；
- 把 `state.count` 当参数传给 composable = 传的是那一刻的数值 → 断链。

面试时按这个顺序讲，比罗列 API 差异高一个档次。下面逐条把"断在哪、怎么救、救的时候有什么代价"讲清楚。

## 核心知识点

### 1. 解构：断的到底是哪个部分（很多人答错）

```js
import { reactive, toRefs, toRef } from 'vue'

const state = reactive({ count: 0, nested: { a: 1 } })

// ❌ 原始值：断链
const { count } = state
state.count++
console.log(count)        // 0 —— 解构那一刻的快照

// ⚠️ 嵌套对象：其实没断
const { nested } = state
state.nested.a = 99
console.log(state.nested.a)  // 99 —— nested 本身就是个 proxy
```

**关键区别在这里**：解构会"读一次属性"，然后把这个读到的值赋给新变量。

- 如果读到的是**原始值**（number/string/bool），拿到就是个普通值，和源对象再无关系；
- 如果读到的是**对象**，reactive 是深层代理，`state.nested` 取出来本来就是同一个嵌套代理对象——**访问它的属性依然走 proxy，依然被追踪**。

所以准确的说法是：**解构断掉的是"这个本地绑定和源属性之间的连接"，而不是"整个值都不响应了"。** 面试里能说出这层区分，说明你真理解代理的粒度。

救法就是别让"绑定"变成普通值，用 `toRefs` / `toRef` 保持连接：

```js
const state = reactive({ count: 0, name: 'Ada' })

// ✅ toRefs：批量转 ref 后再解构（适合 return 一整个 state）
const { count, name } = toRefs(state)
count.value++            // 等价于 state.count++，双向同步

// ✅ toRef：单个属性（适合当参数传给 composable）
const countRef = toRef(state, 'count')
```

`toRef` 里有个 `toRefs` 做不到的能力常被考到：

```js
// ✅ toRef 对"目前还不存在的属性"也能用：返回一个可用的 ref，写入时会补上这个属性
const status = toRef(state, 'status')
status.value = 'open'
console.log(state.status)   // 'open'

// ❌ toRefs 只对"调用时已存在且可枚举"的属性生成 ref（实测）
const s = reactive({ a: 1 })
const refs = toRefs(s)
const b = toRef(s, 'b')
b.value = 2
console.log('b' in refs)    // false —— 后加的属性不会出现在之前生成的 toRefs 里
```

另外 3.3+ 起，`toRef` 支持 **getter 形态**，官方推荐用在 composable 传 props 上：

```ts
// 拿 props.foo 的实时值传给 composable（props 不允许直接改写，所以是只读的活引用）
const props = defineProps<{ foo: string }>()
useSomeFeature(toRef(() => props.foo))   // ✅ 3.3+ 推荐写法
useSomeFeature(toRef(props, 'foo'))      // 老写法，等价
```

这里有个必须点明的限制：**`toRef(props, 'foo')` 得到的是可写 ref，但对 props 写仍然是不允许的**（等价于直接改 prop，Vue 会警告）。需要读写就用 `computed({ get, set })` 显式声明。

### 2. `reactive` 的三条硬限制，决定了它该用在哪儿

```js
// ❌ 限制一：只接受对象。实测对基础类型会警告并原样返回：
// [Vue warn] value cannot be made reactive: 0
const n = reactive(0)   // n === 0，完全没响应式

// ❌ 限制二：不能整体替换（变量换对象 = 断链）
let state = reactive({ count: 0 })
state = reactive({ count: 1 })   // 旧依赖还挂在旧代理上；后续更新与新变量无关
```

实测确认"断链"有多彻底：我建了一个 `watchEffect` 追踪 `state.count`，然后把 `state` 变量换成新的 `reactive({ count: 1 })`，再改新对象的 `count`——**effect 一次都没重跑**。因为 effect 里闭包捕获的是那个变量名，变量换了指向，但副作用本来就是靠旧的 proxy 收集的依赖。

整体替换的正确写法有两种：

```js
// ✅ 改属性，别换对象
Object.assign(state, { count: 1 })

// ✅ 或者一开始就用 ref 包一层，通过 .value 换（推荐容器）
const state = ref({ count: 1 })
state.value = { count: 2 }   // 触发 ref 的 setter，正常通知
```

`ref` 能整体替换，是因为**替换动作发生在 `.value` 的 setter 上**，那条链路是在被追踪的。

**限制三：深层代理 + 不能替换，共同导致官方"优先用 `ref`"的建议。** 但 `reactive` 也不是不能用，它有个 `ref` 比不了的优势：不用写 `.value`，物理解构（`state.list` 而不是 `form.value.list`）。

| 场景 | 推荐 |
|---|---|
| 基础类型 / 会被整体替换的状态 | `ref` |
| 表单这样成组、字段多、内部互相引用的对象 | `reactive` |
| 需要解构或传给函数的状态 | `ref` 或 `toRefs(state)` |
| 大对象 / 第三方实例，只换指针 | `shallowRef` |

### 3. `ref` 的自动解包：四条规则，一条都不能省

"ref 会自动解包"这句话只对了一半，完整规则是四条（全部实测过）：

| 位置 | 是否解包 | 说明 |
|---|---|---|
| **模板里的顶层属性** | ✅ 解包 | `setup` 返回的对象会被 `proxyRefs` 浅层解包 |
| **`reactive` 对象的属性** | ✅ 解包 | `obj.c === c.value` 为 `true`，且双向同步 |
| **赋值给 `reactive` 属性时** | ✅ 写入时解包 | `holder.rc = rc` 后 `typeof holder.rc === 'number'`，改原 ref 会同步过去 |
| **数组元素 / Map、Set 等原生集合元素** | ❌ **不解包** | 必须写 `books[0].value`、`map.get('count').value` |
| **`shallowReactive` 的属性** | ❌ 不解包 | 浅层代理不处理 ref 解包 |

```js
const count = ref(1)
const obj = reactive({ count })
console.log(obj.count === count.value)   // true —— 解包

// ❌ 数组 / Map 里不解包（实测 isRef() 为 true，必须 .value）
const books = reactive([ref('Vue 3 Guide')])
console.log(books[0].value)

const map = reactive(new Map([['count', ref(0)]]))
console.log(map.get('count').value)
```

**模板里那条最容易被"看起来正常"骗到。** 实测两个写法：

{% raw %}
```text
{{ object.id }}       → 渲染成 1        // 看起来正常
{{ object.id + 1 }}   → 渲染成 [object Object]1   // 露馅了
```
{% endraw %}

原因是：模板解包只作用于**渲染上下文的顶层属性**（这一步靠 `proxyRefs` 实现，浅层）。`object` 是顶层所以解包了，但 `object.id` 不是——它仍然是个 ref 对象。单独插值时，Vue 的 `toDisplayString` 内部有个 replacer 会把 ref 显示成 `.value`，所以**看起来是对的**；一旦参与运算（`+ 1`）就变成对象拼接。

所以正确写法是让它保持顶层，或者显式 `.value`：

{% raw %}
```vue
<script setup>
const object = { id: ref(1) }
</script>

<template>
  <!-- ✅ 显式取值 -->
  <p>{{ object.id.value }}</p>
</template>
```
{% endraw %}

（`ref` 包对象时不用管：`const deep = ref({ n: 1 })`，模板里 `{ { deep.n } }` 是通的——因为 `deep` 是顶层、被解包了，`.n` 走的是解包后的普通对象属性访问，追踪正常。实测更新 `deep.value.n` 会重渲染。）

### 4. 重新赋值这件事，`ref` 也有一半会断

很多人知道"`reactive` 不能整体替换"，但**漏了 `ref` 也有同样的坑**——注意区分是替换 `.value` 还是替换**变量本身**：

```js
let count = ref(0)

// ❌ 替换变量本身：断链（实测 effect 一次都不重跑）
count = ref(1)

// ✅ 替换 .value：正常触发
count.value = 1
```

实测里我做过对照：建 `watchEffect` 追踪 `count.value` 后，把 `count` 变量换成新的 `ref(1)` 再改 `.value`，**effect 完全没反应**。原因和第 2 节一模一样：依赖收集在旧的 ref 对象上，变量换指向不影响旧对象。

**规律可以合并成一句**：*任何"换指向"的操作都会断链，只有"改被代理对象内部的属性"才安全。*

### 5. `shallowRef` / `shallowReactive` / `triggerRef`

浅层响应式的语义很简单：**只追踪最外层**。

```js
const big = shallowRef({ n: 0 })
big.value.n = 1        // ❌ 不触发（不追踪深层）
big.value = { n: 2 }   // ✅ 触发（换 .value）
triggerRef(big)        // ✅ 强制触发一次（逃生舱）

const s = shallowReactive({ count: 0, nested: { a: 1 } })
s.count++              // ✅ 触发
s.nested.a++           // ❌ 不触发
```

实测三连（同一份 `watchEffect`）：

| 操作 | effect 是否重跑 |
|---|---|
| `big.value.n = 1`（深层改动） | 否 |
| `triggerRef(big)` | **是**（手动补一次） |
| `big.value = { n: 2 }`（整体替换） | 是 |

什么场景该用：**大对象/大数组只整体替换、第三方实例（图表、地图、Three.js 场景）、你不需要追踪内部每个字段的数据**。深层代理的开销是实打实的——实测同一份嵌套数据做 30 万次属性读取：深层 `reactive` 约 35ms，`shallowRef` 约 2ms，**差一个数量级**（具体毫秒数随机器波动，看数量级就行）。

两个容易答错的点：

- **`triggerRef` 不是常规手段，是逃生舱**。能用"整体替换 `.value`"就整体替换；`triggerRef` 会让所有依赖这个 ref 的副作用一起重跑，粒度很粗。
- **`shallowRef(alreadyRef)` 会直接把原 ref 返回**：实测 `shallowRef(inner) === inner` 为 `true`（`ref(inner) === inner` 也是 `true`）。这是 `createRef` 里的分支——传进去的已经是 ref 就原样返回。所以"用 `shallowRef` 包一层以关掉深层追踪"对一个已经是 ref 的值是无效的。

### 6. 状态所有权：composable 里的 ref 写在哪一行，天差地别

这不属于 Vue 的 API 规则，而是 JS 作用域——但它是"响应式不生效"类 bug 的高发区：

```js
// ❌ 模块级：全局单例，所有组件共享同一份状态
const sharedCount = ref(0)
export function useCounterShared() {
  return { sharedCount }
}

// ✅ 函数内：每次调用独立，跟组件实例同生命周期
export function useCounterLocal() {
  const count = ref(0)
  return { count }
}
```

判断方法一句话：**`ref()` 这一行在函数外面 → 共享；在函数里面 → 独立。** 想要全局共享就用第一种（Pinia 也是这个原理），但别误以为它是"每次调用一份"。

顺带两个相邻的高频点：

- **Pinia store 不能直接解构**：`const { count } = useStore()` 拿到的同样是快照。要用 `storeToRefs(store)`——它是 Pinia 版的 `toRefs`，而且**它还负责保住 getters 的响应性**（解构 actions 不会丢，因为 action 是普通函数）。
- **Nuxt / SSR 场景用 `useState()`**：它的作用等价于 `ref`，但能把服务端的状态正确传给客户端，避免 hydration 不一致。

## 其实你每天都在用

- 把 `reactive` 的表单对象 `const { name, age } = form` 解构给子组件，然后发现输入框不动了——就是第 1 节那个坑。
- `v-model` 的 `defineModel()` 返回的本质是个 ref，所以模板里写 `v-model="model"` 不用 `.value`，脚本里必须写。
- `computed(() => list.value.filter(...))`——computed 返回的也是 ref（`isRef()` 为 `true`），所以模板自动解包、脚本要 `.value`。
- 图表、地图这类实例用 `shallowRef` 存：实例内部几万个属性，深层代理纯属浪费，而且你从来不会改它内部字段。
- 从 Pinia 里 `storeToRefs(store)` 解构状态——直接解构是新手最容易踩的坑之一。
- `toRefs(props)` 在组件里 `const { title } = toRefs(props)`，比到处写 `props.title` 干净一点，且保留响应性。
- 传参数给 composable 时写 `useFetch(toRef(() => props.id))`，这是 3.3 之后官方推荐的写法。
- `ref` 里嵌套 `ref` 会自动深度解包（实测 `typeof outer.value.inner === 'number'`），所以别指望在 ref 里"存一个 ref 本身"。

## 常见误解（FAQ）

**❌ 误区1："解构 reactive 出来的数据全部丢失响应式"**

不准确。丢的是**原始值**（number/string/boolean 这些，解构出来是快照）。如果解构出来的是**对象**，它本身就是深层代理，访问它的属性照样被追踪——实测 `const { nested } = state` 后改 `state.nested.a`，读 `nested.a` 是最新值。准确表述：**解构断掉的是"本地绑定 ↔ 源属性"这条连接**，不是把值变成普通值。

**❌ 误区2："reactive 不能整体替换，ref 可以随便替换"**

前半对，后半有坑。**替换 ref 变量本身（`count = ref(1)`）同样断链**，实测 effect 一次都不重跑。只有 `count.value = 1` 才是安全的。统一规律：**换指向必断，改属性才安全。**

**❌ 误区3："ref 在任何地方都会自动解包"**

只对"模板顶层属性"和"`reactive` 对象的属性"成立。**数组元素、`Map` / `Set` 等原生集合元素不解包**（实测 `isRef(books[0])` 为 `true`，必须 `.value`）；**`shallowReactive` 的属性也不解包**。另外模板里嵌套的 ref 也不解包——`{ { object.id + 1 } }` 会渲染成 `[object Object]1`。

**❌ 误区4："`toRef` 和 `toRefs` 只是批量/单个的区别"**

还有一个关键差异：**`toRef` 对当前不存在的属性也能返回可用 ref，写入时会把属性补上；`toRefs` 只对调用时已存在且可枚举的属性生成 ref**（实测后加的键不会出现在之前生成的 `toRefs` 里）。处理可选字段、动态字段时必须用 `toRef`。

**❌ 误区5："`reactive(0)` 会把它变成响应式数字"**

不会。`reactive` **只能接收对象类型**，实测传基础类型会打印 `[Vue warn] value cannot be made reactive: 0` 并**原样返回 0**——你拿到的是个普通数字，毫无响应性。单值请用 `ref`。

**❌ 误区6："`shallowRef` 改深层属性后 `triggerRef` 是官方推荐做法"**

`triggerRef` 是**逃生舱不是常规手段**——它会让所有依赖这个 ref 的副作用一起重跑，粒度很粗。官方给的主线用法是"**整体替换 `.value`**"。另外实测 `shallowRef(已有的ref) === 那个ref`（`ref(已有的ref)` 同理），所以拿它包一个已经是 ref 的值来"关掉深层追踪"是没效果的。

**❌ 误区7："`storeToRefs` 只是为了少写 `store.`"**

它的真正作用是**保住响应性**。直接 `const { count } = useStore()` 解构出来的是快照（跟解构 `reactive` 一个道理），而 `storeToRefs` 把 state 和 getters 都转成关联的 ref，所以你改 `.value` 能写回 store、getter 也仍然跟着变。注意它**只处理 state 和 getters**，actions 不需要包（函数解构不丢东西）。

**❌ 误区8："对象用 reactive 性能更好，因为 ref 多包了一层"**

不成立。实测 `ref({ a: 1 }).value` 本身就是一个 `reactive` 代理（`isReactive()` 为 `true`）——**`ref(对象)` 内部走的就是 `reactive` 那套深层代理，没有额外的拷贝层**。两者的真实差别是"能不能整体替换"和"要不要写 `.value`"，不是性能。

## 一句话总结

**Vue 只追踪"通过 proxy 的读写"：解构、换指向、传数值都会断链，能救的只有 `ref` / `toRef` / `toRefs` 这类"把连接保住"的写法——记住"换指向必断，改属性才安全"，八成的响应式失效 bug 都能当场定位。**
