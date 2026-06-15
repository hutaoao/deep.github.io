---
layout: post
title: "computed与watch原理深度解析"
date: 2026-02-22
tags: [Vue3, computed, watch, 响应式]
categories: [前端核心, Vue]
---

## 一句话概括

`computed` 是**惰性的派生值** —— 它基于依赖缓存计算结果，仅在依赖变化且自身被读取时才重新求值；`watch` 是**响应式的副作用触发器** —— 它监听响应式数据的变化，在变化发生时执行用户定义的回调函数，并支持多种时序控制（`flush`）、深度监听（`deep`）和立即执行（`immediate`）。两者共享 Vue 3 的 `effect` 响应式底层，但设计目标、执行时机和缓存策略截然不同。

## 背景与意义

### 为什么需要 computed 和 watch？

在 Vue 2 时代，开发者经常在模板中编写复杂的表达式：

{% raw %}
```html
<!-- Vue 2 模板中的复杂表达式 -->
<div>{{ message.split('').reverse().join('') }}</div>
```
{% endraw %}

这种做法不仅让模板臃肿不堪，更会导致每次渲染都重复执行昂贵的计算逻辑。`computed` 正是为解决这一问题而生——它提供了一种**声明式派生状态**的机制，让开发者能够将复杂计算逻辑封装起来，同时自动追踪依赖并缓存结果。

而 `watch` 则解决了另一类问题：当数据变化时，我们需要执行**副作用**——比如发起 API 请求、操作 DOM、写入 localStorage、同步状态到外部系统等。这些操作不能放在 `computed` 中，因为 `computed` 应该是纯函数（无副作用），并且不应该依赖其返回值被消费的时机。

### 从 Vue 2 到 Vue 3 的演进

Vue 3 的响应式系统基于 `Proxy` 重写，不再是 Vue 2 的 `Object.defineProperty` + 依赖数组。这带来了两个根本性变化：

1. **`computed` 可以正确处理数组索引、属性动态增删**
2. **`watch` 不再需要 `deep` 选项也能监听到深层属性变化**（虽然 `deep` 仍然控制着遍历和回调触发的行为）

更重要的是，Vue 3 将 `effect` 作为统一底层抽象 —— `computed`、`watch`、`watchEffect`、渲染器本身，全都建立在 `ReactiveEffect` 之上。

## 概念与定义

### computed（计算属性）

`computed` 是一个基于其响应式依赖进行**缓存**的计算值。它接受一个 getter 函数，或者一个包含 `get` 和 `set` 的对象。

```typescript
// 函数签名（简化）
export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>,
  debuggerOptions?: DebuggerOptions
): ComputedRef<T>
```

核心特征：
- **惰性执行**：只在被读取时才计算
- **缓存机制**：依赖不变时返回缓存值
- **自动追踪**：自动收集 getter 中访问的响应式依赖
- **可写版本**：通过 `set` 实现双向绑定

### watch（侦听器）

`watch` 监听一个或多个响应式数据源，当数据源变化时执行回调函数。

```typescript
// 函数签名（简化）
export function watch<T, Immediate extends Readonly<boolean> = false>(
  source: WatchSource<T> | WatchSource<T>[],
  cb: WatchCallback<T>,
  options?: WatchOptions
): WatchStopHandle
```

核心特征：
- **懒执行**：默认只在数据变化时触发（除非设置 `immediate: true`）
- **灵活的数据源**：支持 ref、reactive 对象、getter 函数、数组组合
- **回调参数丰富**：`(newValue, oldValue, onCleanup)` 提供更新前后的值
- **副作用清理**：通过 `onCleanup` 支持竞态处理

### watchEffect

`watchEffect` 是 `watch` 的简化版——它立即执行传入的函数，并自动追踪函数中访问的所有响应式依赖。

```typescript
export function watchEffect(
  effect: (onCleanup: OnCleanup) => void,
  options?: WatchOptionsBase
): WatchStopHandle
```

与 `watch` 的核心区别：
- **自动确定依赖**：不需要显式指定数据源
- **立即执行**：初始化时立即运行一次以收集依赖
- **无新/旧值**：回调不接收 `newValue` 和 `oldValue`

## 最小示例

### computed 基础用法

```typescript
import { ref, computed } from 'vue'

const count = ref(0)

// 只读计算属性
const doubled = computed(() => count.value * 2)

console.log(doubled.value) // 0
count.value = 5
console.log(doubled.value) // 10

// 可写计算属性
const writable = computed({
  get: () => count.value + 1,
  set: (val) => { count.value = val - 1 }
})
writable.value = 100
console.log(count.value) // 99
```

### watch 基础用法

```typescript
import { ref, watch } from 'vue'

const question = ref('')
const answer = ref('')

// 基础侦听
watch(question, async (newQ, oldQ) => {
  if (newQ.includes('?')) {
    const res = await fetch('https://api.example.com/ask/' + newQ)
    answer.value = await res.text()
  }
})

// 侦听多个数据源
watch([question, answer], ([newQ, newA], [oldQ, oldA]) => {
  console.log(`question: ${oldQ} → ${newQ}, answer: ${oldA} → ${newA}`)
})
```

### watchEffect 基础用法

```typescript
import { ref, watchEffect } from 'vue'

const room = ref('living-room')
const temp = ref(22)

// 立即执行，自动追踪依赖
watchEffect(() => {
  // 访问 room.value 和 temp.value 时，自动注册依赖
  console.log(`${room.value} temperature is ${temp.value}°C`)
})
// 立即输出: "living-room temperature is 22°C"

temp.value = 25
// 输出: "living-room temperature is 25°C"
```

## 核心知识点拆解

### 1. computed 的懒执行与缓存机制

Vue 3 的 `computed` 最精妙的设计在于**懒执行（lazy evaluation）**。与 Vue 2 不同的是，Vue 3 的计算属性不会在创建时立即执行 getter，而是在第一次访问 `.value` 时才真正执行。

缓存机制通过 **dirty 标志位** 实现，算法流程如下：

```
访问 computed.value
  ├─ dirty === false → 返回缓存值 _value（不执行 getter）
  └─ dirty === true  → 执行 getter 更新 _value，设置 dirty = false，返回 _value

依赖变更通知到达
  └─ 仅设置 dirty = true（不立即重新求值！）
```

这一设计带来了显著的性能优势：

{% raw %}
```typescript
const bigList = ref(generateHugeArray(100000))
const expensive = computed(() => expensiveTransform(bigList.value))

// 场景一：模板中多处使用
<template>
  <div>{{ expensive }}</div>  <!-- 第一次读取：计算 -->
  <div>{{ expensive }}</div>  <!-- 缓存命中 -->
  <div>{{ expensive }}</div>  <!-- 缓存命中 -->
</template>

// 场景二：只在特定条件下使用
if (showDetails.value) {
  console.log(expensive.value)  // 只有这里才真正计算
}
```
{% endraw %}

### 2. computed dirty 标志位的实现原理

`dirty` 标志位是 computed 缓存的核心。来看源码中的关键逻辑：

```typescript
// Vue 3 源码 - packages/reactivity/src/computed.ts（简化）
export class ComputedRefImpl<T> {
  public dep?: Dep
  private _value!: T
  public readonly effect: ReactiveEffect<T>

  // dirty 标志位：true 表示需要重新计算
  private _dirty = true

  constructor(getter: ComputedGetter<T>) {
    // 核心：创建一个 ReactiveEffect
    this.effect = new ReactiveEffect(getter, () => {
      // 调度器：当依赖变化时触发
      if (!this._dirty) {
        this._dirty = true
        // 触发该 computed 自己的依赖（订阅者）更新
        triggerRefValue(this)
      }
    })
    this.effect.computed = this
  }

  get value() {
    // 读取时收集当前 activeEffect 作为依赖
    trackRefValue(this)
    // 仅在 dirty 时重新求值
    if (this._dirty) {
      this._dirty = false
      // run() 内部会 pushEffect/popEffect 并执行 getter
      this._value = this.effect.run()!
    }
    return this._value
  }

  set value(newValue: T) {
    // 只有 writable computed 才需要实现 set
    // ...
  }
}
```

**dirty 标志位的完整生命周期：**

1. **初始状态**：`_dirty = true`，第一次 `get value()` 时执行 getter
2. **缓存状态**：getter 执行后 `_dirty = false`，后续访问直接返回 `_value`
3. **依赖变化**：getter 内的响应式数据变化 → 触发调度器 `scheduler` → `_dirty = true` → `triggerRefValue`
4. **重新计算**：下一次 `get value()` 时检测到 dirty → 重新执行 getter → 回到步骤 2

这里有一个关键的优化细节：**依赖变化时只设置 dirty 标志，并不立即求值**。这意味着即使一个 computed 的依赖在一帧内变化了 100 次，它也只会重新计算一次（在真正被读取的时候）。

### 3. watch 的 flush 时机

`watch` 的第三个参数 `flush` 控制回调的执行时机，有三种模式：

```typescript
interface WatchOptions {
  flush?: 'pre' | 'post' | 'sync'  // 默认 'pre'
}
```

| flush 模式 | 执行时机 | 适用场景 |
|-----------|---------|---------|
| `pre`（默认） | 组件更新前，同步队列中 | 需要在 DOM 更新之前访问旧状态 |
| `post` | 组件更新后，`nextTick` 之后 | 需要在 DOM 更新后操作 DOM |
| `sync` | 数据变化时立即执行 | 需要实时同步，开启性能警告 |

**`pre` 模式**（默认）：
```typescript
// watch 默认使用 'pre' flush
watch(source, callback)
// 等价于
watch(source, callback, { flush: 'pre' })
```
在 `pre` 模式下，回调会在组件 **re-render 之前** 执行。这意味着如果回调中修改了其他响应式数据，这些修改会合并到当前渲染周期中，不会触发额外的渲染。

**`post` 模式**：
```typescript
watch(source, callback, { flush: 'post' })
```
此时回调在组件完成 DOM 更新后执行，类似于 `watch + nextTick`。常用于需要访问更新后 DOM 的场景：

```typescript
const el = ref<HTMLElement>()
const data = ref([1, 2, 3])

watch(data, () => {
  // 此时 DOM 已经更新，可以测量元素尺寸
  console.log(el.value?.scrollHeight)
}, { flush: 'post' })
```

**`sync` 模式**：
```typescript
watch(source, callback, { flush: 'sync' })
```
数据变化时立即执行回调，没有任何去抖或批处理。Vue 官方文档提示，**除非必要，否则不要使用 `sync`**，因为它会破坏响应式系统的批处理优化，可能导致频繁的更新和无限循环。

**Vue 3.4+ 的 `flush: 'pre'` 和 `onWatcherCleanup`：**

在 Vue 3.4 引入了 `onWatcherCleanup` API，使 watch 内部的竞态处理更加优雅。当 watch 回调在 `flush: 'pre'` 模式下执行时，Vue 保证了所有 `pre` 侦听器按依赖顺序执行，这解决了 Vue 2 中令人头疼的跨组件 watch 执行顺序问题。

### 4. watch 的 deep 选项实现

```typescript
watch(obj, callback, { deep: true })
```

`deep: true` 的实现核心是**递归遍历**响应式对象，使其内部所有嵌套属性都被当前 `effect` 收集为依赖。

```typescript
// Vue 3 源码 - packages/reactivity/src/apiWatch.ts（简化）
function traverse(value: unknown, seen: Set<unknown> = new Set()): unknown {
  // 防止循环引用
  if (!isObject(value) || seen.has(value)) {
    return value
  }
  seen.add(value)

  // 如果是数组，遍历每个元素
  if (isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      traverse(value[i], seen)
    }
  } else if (isObject(value)) {
    // 如果是对象，遍历所有属性
    for (const key in value) {
      traverse(value[key], seen)
    }
  }
  return value
}
```

关键原理：当 `watch` 的 `deep` 为 `true` 时，在执行 `getter`（获取当前值）之前，Vue 会对返回值调用 `traverse` 函数，**递归访问**对象的所有嵌套属性。因为响应式系统的依赖收集机制，访问属性即意味着注册依赖——所有访问过的深层属性都成为该 watch 的依赖。

```typescript
// 实际在 Vue 3 源码中的处理
const getter = () => {
  // 获取 source 的当前值
  const value = isRef(source)
    ? source.value
    : isReactive(source)
      ? source
      : source()

  // 如果 deep: true，递归遍历以收集所有深层依赖
  if (cb && deep) {
    traverse(value)
  }
  return value
}
```

**性能注意**：`deep: true` 是 O(n) 操作——每次依赖变化时都要遍历整个对象树。对于大型深层对象（如 10,000+ 节点），这会带来可感知的性能开销。更好的做法是使用精确的 getter 函数代替 `deep`：

```typescript
// ❌ 低效：deep 遍历整个对象
watch(obj, callback, { deep: true })

// ✅ 高效：精确指定需要侦听的路径
watch(() => obj.a.b.c, callback)

// ✅ 更高效：侦听多个精确路径
watch(
  [() => obj.a.b, () => obj.x.y],
  ([newB, newY], [oldB, oldY]) => { ... }
)
```

### 5. watch immediate 选项

```typescript
watch(source, callback, { immediate: true })
```

`immediate: true` 的行为很简单：初始化时**立即执行一次 callback**。此时 `oldValue` 的值为 `undefined`。

源码实现的核心逻辑：

```typescript
// 源码简化
if (immediate) {
  // 立即执行一次回调，oldValue 为 undefined
  cbImmediate = (newVal: T, oldVal: T) => cb(newVal, undefined, onCleanup)
} else {
  cbImmediate = null
}

// 在 effect 初始化后
if (cbImmediate) {
  // 首次调用回调
  cbImmediate(newValue, undefined)
}
```

一个实际场景是初始化时加载数据：

```typescript
const userId = ref(1)

// 没有 immediate：需要手动调用一次
const fetchUser = async (id: number) => {
  const res = await fetch(`/api/users/${id}`)
  user.value = await res.json()
}
fetchUser(userId.value)  // 手动初始化
watch(userId, (id) => fetchUser(id))

// 有 immediate：watch 自动处理初始化
watch(userId, async (id, oldId) => {
  await fetchUser(id)
}, { immediate: true })
```

### 6. watchEffect 与 watch 的区别

| 维度 | `watch` | `watchEffect` |
|-----|---------|--------------|
| 数据源 | 显式指定 | 自动追踪 |
| 初始执行 | 否（除非 `immediate: true`） | 是 |
| 回调参数 | `(newVal, oldVal, onCleanup)` | `(onCleanup)` |
| 访问旧值 | ✅ 有 `oldValue` | ❌ 无 |
| 精确控制 | ✅ 可监听单个属性 | ❌ 自动收集全部 |
| 适用场景 | 需要新旧值对比、精确依赖 | 被动副作用、日志、同步状态 |

**什么时候用 watchEffect 代替 watch？**

```typescript
// watch：需要显式列出依赖
watch([room, temp], () => {
  saveToLocalStorage({ room: room.value, temp: temp.value })
})

// watchEffect：自动追踪，代码更简洁
watchEffect(() => {
  saveToLocalStorage({ room: room.value, temp: temp.value })
})
```

**`watchPostEffect` 和 `watchSyncEffect` 是 Vue 3 提供的别名：**

```typescript
// 等价于 watchEffect 的 flush: 'post'
import { watchPostEffect } from 'vue'

// 等价于 watchEffect 的 flush: 'sync'
import { watchSyncEffect } from 'vue'
```

## 实战案例

### 案例一：购物车价格计算（computed 经典场景）

```typescript
import { ref, computed, reactive } from 'vue'

interface CartItem {
  id: number
  name: string
  price: number
  quantity: number
}

const cart = reactive<CartItem[]>([
  { id: 1, name: 'Vue.js 实战', price: 79, quantity: 2 },
  { id: 2, name: '算法导论', price: 128, quantity: 1 },
  { id: 3, name: '机械键盘', price: 399, quantity: 1 },
])

// 多个计算属性组合，每个都各自缓存
const subtotal = computed(() =>
  cart.reduce((sum, item) => sum + item.price * item.quantity, 0)
)

const discount = computed(() => {
  // 满 300 减 30
  return subtotal.value >= 300 ? 30 : 0
})

const shipping = computed(() => {
  // 满 200 包邮，否则收 20 运费
  return subtotal.value >= 200 ? 0 : 20
})

const total = computed(() => subtotal.value - discount.value + shipping.value)

const itemCount = computed(() =>
  cart.reduce((sum, item) => sum + item.quantity, 0)
)

// 当修改购物车时，只有 total 链上的 computed 被标记为 dirty
// 但只有被访问的那个才会真正重新计算
cart[0].quantity = 3
console.log(total.value) // 触发 subtotal → discount/shipping → total 的链式计算
```

### 案例二：搜索防抖 + 自动请求（watch + watchEffect）

```typescript
import { ref, watch, watchEffect, computed } from 'vue'

const searchQuery = ref('')
const searchResults = ref<string[]>([])
const isLoading = ref(false)

// 1. 搜索输入防抖
const debouncedQuery = ref('')
let debounceTimer: ReturnType<typeof setTimeout>

watch(searchQuery, (newVal) => {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    debouncedQuery.value = newVal
  }, 300)
})

// 2. 发起搜索请求
watch(debouncedQuery, async (query, oldQuery, onCleanup) => {
  if (!query.trim()) {
    searchResults.value = []
    return
  }

  isLoading.value = true
  let cancelled = false

  // 注册清理函数：如果用户在请求完成前再次修改了查询
  onCleanup(() => {
    cancelled = true
  })

  try {
    const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`)
    const data = await res.json()

    if (!cancelled) {
      searchResults.value = data.results
    }
  } finally {
    if (!cancelled) {
      isLoading.value = false
    }
  }
})

// 3. 日志记录：自动追踪所有搜索相关状态
watchEffect(() => {
  if (searchQuery.value) {
    console.log(`Search: "${searchQuery.value}" → ${searchResults.value.length} results, loading=${isLoading.value}`)
  }
})
```

### 案例三：可写 computed 实现双向绑定

```typescript
import { ref, computed } from 'vue'

// 一个完整的名字拆分与合并
const firstName = ref('John')
const lastName = ref('Doe')

// 可写 computed：允许外部通过 fullName.value = 'Jane Smith' 来设置
const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (val) => {
    const parts = val.split(' ')
    firstName.value = parts[0] || ''
    lastName.value = parts.slice(1).join(' ') || ''
  }
})

console.log(fullName.value) // "John Doe"
fullName.value = 'Jane Smith'
console.log(firstName.value) // "Jane"
console.log(lastName.value)  // "Smith"
```

可写 computed 的 `set` 中不能直接修改 computed 自身（会触发 set 无限递归），而是必须修改它的底层依赖——这正是设计用意所在。

### 案例四：watch 监听路由变化

```typescript
import { watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const pageData = ref(null)

// 监听整个 route 对象（reactive 对象，默认 deep）
watch(route, (to, from) => {
  // 路由变化时重新获取页面数据
  fetchPageData(to.path, to.query)
}, { immediate: true })

// 更精确：只监听 params 变化
watch(() => route.params.id, async (newId, oldId) => {
  if (newId !== oldId) {
    pageData.value = await fetch(`/api/page/${newId}`).then(r => r.json())
  }
})
```

**性能提示**：整个 `route` 对象是 reactive 的，`watch(route, cb)` 默认是 `deep` 监听（reactive 对象的 watch 天然有 deep 行为）。如果只需要特定路径参数变化，使用 getter 函数更高效。

## 底层原理

### ReactiveEffect：一切响应式的基石

Vue 3 中 `computed`、`watch`、`watchEffect` 的底层都是 `ReactiveEffect` 类的实例。这个抽象的诞生是 Vue 3 相比 Vue 2 最大的架构进步之一。

```typescript
// 核心类（概念性简化）
class ReactiveEffect<T = any> {
  active = true
  deps: Dep[] = []          // 当前 effect 注册了哪些依赖
  computed?: boolean        // 标记是否为 computed

  constructor(
    public fn: () => T,          // 实际要执行的函数
    public scheduler?: () => void  // 调度器（watch/computed 专属）
  ) {}

  run() {
    // 将自身设为 activeEffect
    activeEffect = this
    // 执行 fn，期间访问的响应式数据会自动调用 track(this)
    const result = this.fn()
    // 恢复上一个 activeEffect
    activeEffect = this.parent
    return result
  }
}
```

**三个角色的 effect 配置差异：**

```
                    ┌─────────────┬──────────────┬──────────────┐
                    │  computed   │    watch     │  watchEffect  │
├─────────────┼──────────────┼──────────────┤
│ scheduler   │  设置dirty   │   执行回调    │    执行fn     │
│ lazy        │     true     │    false     │    false      │
│ 首次执行    │  访问时触发   │  立即(或immediate) │  立即       │
│ 返回值      │  缓存值      │  stop句柄     │  stop句柄     │
└─────────────┴──────────────┴──────────────┘
```

### computedRefImpl 完整源码分析

下面是对 Vue 3 `computed.ts` 核心实现的逐行解读（基于 Vue 3.4+）：

```typescript
// packages/reactivity/src/computed.ts
export class ComputedRefImpl<T> {
  // 该 computed 的订阅者（依赖它的 watch/render/computed 等）
  public dep?: Dep

  // 缓存的当前值
  private _value!: T

  // 核心 ReactiveEffect 实例
  public readonly effect: ReactiveEffect<T>

  // 脏值检查标志——缓存的开关
  private _dirty = true

  // 缓存是否已被读取过（用于调试和 __v_isRef 判断）
  public readonly __v_isRef = true
  public readonly [ReactiveFlags.IS_READONLY]: boolean

  constructor(
    getter: ComputedGetter<T>,
    private readonly _setter: ComputedSetter<T>,
    isReadonly: boolean
  ) {
    // 创建 ReactiveEffect
    // 第一个参数：getter——计算值的函数
    // 第二个参数：scheduler——当依赖变化时触发的调度函数
    this.effect = new ReactiveEffect(getter, () => {
      // === 调度器开始 ===
      // 当 computed 的任一依赖变化时，以下代码执行

      // 如果当前已经是 dirty，不需要重复设置
      if (!this._dirty) {
        this._dirty = true

        // 触发该 computed 自己的订阅者
        // 如果有组件或其他 computed 依赖于这个 computed
        triggerRefValue(this)
      }
      // === 调度器结束 ===
    })

    // 双向引用：effect 持有 computed 的引用
    this.effect.computed = this
    this[ReactiveFlags.IS_READONLY] = isReadonly
  }

  get value() {
    // 1. 依赖收集：将当前 activeEffect 注册为该 computed 的订阅者
    //    如果有组件正在渲染并访问了此 computed，组件会被注册为依赖
    const self = toRaw(this)
    trackRefValue(self)

    // 2. 脏检查：仅在 dirty 时重新求值
    if (self._dirty) {
      self._dirty = false

      // 3. 执行 effect.run()，内部会：
      //    - 设置 activeEffect = this.effect（暂停外部 effect 收集）
      //    - 执行 getter，此时 getter 内访问的响应式数据会收集 this.effect 作为依赖
      //    - 恢复之前的 activeEffect
      //    - 返回 getter 的执行结果
      self._value = self.effect.run()!
    }

    // 4. 返回缓存值
    return self._value
  }

  set value(newValue: T) {
    this._setter(newValue)
  }
}
```

**巧妙之处在于两点：**

1. **`get value()` 中 `trackRefValue` 在 `run()` 之前**：这意味着访问 computed 的上下文（如组件的渲染 effect）会成为该 computed 的订阅者。而 computed 内部的 getter 在执行时，其中的依赖会将该 computed 的 effect 注册为订阅者。形成了依赖链：`响应式数据 → computed effect → 组件 effect`。

2. **调度器中 `triggerRefValue` 通知的是 computed 的订阅者**：当依赖变化时，调度器将 `_dirty` 设为 `true`，然后通知依赖该 computed 的所有订阅者（如组件渲染函数），这些订阅者会重新执行。当它们再次读取 `computed.value` 时，由于 `_dirty` 为 `true`，就会重新计算。

### 链式 computed 的更新传播

```typescript
const a = ref(1)
const b = computed(() => a.value * 2)
const c = computed(() => b.value + 1)
const d = computed(() => c.value * 10)
```

当 `a.value` 变化时，更新传播路径为：

```
a.value = 2
  → a 的 dep 中通知所有订阅者（包括 b 的 effect）
    → b 的 scheduler 执行：_dirty = true，triggerRefValue(b)
      → b 的订阅者包括 c 的 effect
        → c 的 scheduler 执行：_dirty = true，triggerRefValue(c)
          → c 的订阅者包括 d 的 effect
            → d 的 scheduler 执行：_dirty = true，triggerRefValue(d)
              → d 的订阅者包括渲染 effect
                → 组件重新渲染，访问 d.value
                  → d.dirty → 重新计算，d._value = ?
                    → 访问 c.value → c.dirty → 重新计算
                      → 访问 b.value → b.dirty → 重新计算
                        → 访问 a.value → 2
```

注意：链式传播只设置了 dirty 标志位，没有真正求值。实际求值发生在组件重新渲染时，以深度优先顺序进行。这保证了**每个 computed 最多重新计算一次**，避免了冗余计算。

### watch 的底层实现

`watch` 的实现比 `computed` 更复杂，因为它需要处理多种数据源类型、`immediate`、`deep`、`flush` 等选项。核心源码（简化）：

```typescript
// packages/reactivity/src/apiWatch.ts（简化）
export function watch<T>(
  source: WatchSource<T> | WatchSource<T>[],
  cb: WatchCallback<T>,
  options?: WatchOptions
): WatchStopHandle {
  return doWatch(source, cb, options)
}

function doWatch(
  source: WatchSource | WatchSource[],
  cb: WatchCallback | null,
  options: WatchOptions = {}
): WatchStopHandle {
  const { immediate, deep, flush } = options

  // 1. 根据 source 类型生成 getter 函数
  let getter: () => any
  if (isRef(source)) {
    getter = () => source.value
  } else if (isReactive(source)) {
    getter = () => source
    // reactive 数据源默认 deep
    deep ??= true
  } else if (isArray(source)) {
    getter = () => source.map(s => isRef(s) ? s.value : (isReactive(s) ? s : s()))
  } else if (isFunction(source)) {
    getter = () => source()
  }

  // 2. 处理 deep 选项：递归遍历收集深层依赖
  if (cb && deep) {
    const baseGetter = getter
    getter = () => traverse(baseGetter())
  }

  // 3. 创建 ReactiveEffect
  let oldValue: any = isMultiSource ? [] : INITIAL_WATCHER_VALUE
  const job = () => {
    if (!effect.active) return
    // 获取新值
    const newValue = effect.run()
    // 新旧值比较（使用 Object.is + NaN 处理）
    if (hasChanged(newValue, oldValue)) {
      // 执行回调
      cb(newValue, oldValue, onCleanup)
      oldValue = newValue
    }
  }

  // 4. 调度器：根据 flush 选项控制执行时机
  let scheduler: () => void
  if (flush === 'sync') {
    scheduler = job  // 同步执行
  } else if (flush === 'post') {
    scheduler = () => queuePostRenderEffect(job)  // 渲染后执行
  } else {
    // 'pre'（默认）：在组件更新前执行
    scheduler = () => {
      if (!instance || instance.isMounted) {
        queuePreFlushCb(job)
      } else {
        job()  // 首次挂载前同步执行
      }
    }
  }

  const effect = new ReactiveEffect(getter, scheduler)

  // 5. 处理 immediate
  if (cb && immediate) {
    job()  // 立即执行一次
  } else {
    // 首次运行 effect 收集依赖
    oldValue = effect.run()
  }
}
```

## 高频面试题解析

### Q1：computed 和 method 有什么区别？

**核心区别在于缓存**。computed 基于依赖缓存，依赖不变时访问 `.value` 直接返回缓存结果；而 method 每次调用都会执行函数体。

```typescript
const count = ref(0)

// computed：依赖不变不重新计算
const double = computed(() => {
  console.log('computed evaluated')  // 只在 count 变化时打印
  return count.value * 2
})

// method：每次调用都执行
function doubleFn() {
  console.log('method called')  // 每次调用都打印
  return count.value * 2
}

// 模板中多次使用
// computed: 只输出一次 "computed evaluated"（后续走缓存）
// method: 每次渲染输出三次 "method called"
```

### Q2：computed 的 getter 中能否有副作用？

**技术上可以，但强烈不推荐**。computed 的 getter 在 `dirty = false` 时不会执行，这意味着副作用可能不会被触发；反之如果依赖频繁变化而 computed 恰好被频繁访问，副作用也会频繁触发。副作用应该放在 `watch` 或 `watchEffect` 中。

```typescript
// ❌ 不推荐：副作用在 computed 中
const bad = computed(() => {
  console.log('side effect')  // 依赖不变时不执行
  return someRef.value
})

// ✅ 推荐：副作用独立管理
const result = computed(() => someRef.value)
watchEffect(() => console.log('side effect'))
```

### Q3：watch 和 watchEffect 如何选择？

决策树：
1. **需要访问旧值吗？** → 用 `watch`
2. **数据源需要精确指定吗？** → 用 `watch`
3. **初始化时需要立即执行吗？** → `watch` + `immediate` 或 `watchEffect`
4. **只需要执行副作用不关心旧值？** → `watchEffect`（代码更简洁）
5. **依赖是动态变化的？** → `watchEffect`（自动追踪新依赖）

### Q4：deep: true 会监听新增属性吗？

**会的**。因为 Vue 3 的响应式系统基于 `Proxy`，新增属性同样会被拦截并触发响应。`deep: true` 通过 `traverse` 递归访问对象的所有属性（包括未来新增的），当 `Proxy` 拦截到 `set` 操作时，会触发该 watch 的调度器。

```typescript
const obj = reactive({ a: 1 })

watch(obj, (newVal) => {
  console.log('changed', newVal)
}, { deep: true })

// Proxy 拦截 set
obj.b = 2  // 输出: "changed { a: 1, b: 2 }"
```

### Q5：computed 链式场景下如何避免重复计算？

Vue 3 通过 **dirty 标志位的调度器设计** 天然解决了这个问题。当链式 computed 的根依赖变化时，所有涉及 computed 的 `_dirty` 被设为 `true`，但只在被读取时才重新求值。如果链中某个 computed 没有被读取（如条件渲染），它根本不会计算。

```typescript
const a = ref(1)
const b = computed(() => a.value * 2)  // 昂贵计算
const c = computed(() => a.value * 3)  // 昂贵计算

// 只有 b 和 c 都被使用时才会计算
// 如果模板只用了 b.value，c 不会计算
```

### Q6：flush: 'pre' 和 flush: 'post' 的实际差异是什么？

看一个 DOM 操作的例子：

```typescript
const count = ref(0)

watch(count, (newCount) => {
  // flush: 'pre'（默认）——此时 DOM 尚未更新
  console.log(el.value?.textContent)  // 旧的文本
})

watch(count, (newCount) => {
  // flush: 'post'——DOM 已更新
  console.log(el.value?.textContent)  // 最新的文本
}, { flush: 'post' })

count.value++  // 触发 DOM 更新
```

### Q7：什么是竞态问题？watch 如何处理？

竞态问题（Race Condition）发生在异步操作中：当 watch 回调发起异步请求，如果数据源在请求返回前再次变化，可能导致先发请求的结果覆盖后发请求的结果。

```typescript
// 不处理竞态：可能产生脏数据
watch(query, async (newQ) => {
  const result = await fetch(`/api?q=${newQ}`)
  results.value = await result.json()  // 可能被过时的请求覆盖
})

// 处理竞态：使用 onCleanup
watch(query, async (newQ, oldQ, onCleanup) => {
  let expired = false
  onCleanup(() => { expired = true })  // 新请求发出时，上一个请求被标记过期

  const result = await fetch(`/api?q=${newQ}`)
  if (!expired) {
    results.value = await result.json()
  }
})
```

`onCleanup` 的实现原理：每个 watch 回调维护一个 `cleanup` 变量，在下次回调执行前（或 watch 停止时），调用前一次注册的清理函数。

### Q8：watchEffect 的依赖是如何自动收集的？

`watchEffect(fn)` 的原理非常简单——创建 `ReactiveEffect`，立即调用 `effect.run()`：

```typescript
// 创建 effect 后立即 run
const effect = new ReactiveEffect(fn)
effect.run()  // 执行 fn，期间访问的响应式属性都调用 track(effect)
```

在 `fn` 执行期间，`activeEffect` 被设置为该 effect 实例。当 `fn` 中访问 `someRef.value` 时，`track()` 将 `activeEffect` 注册为 `someRef` 的依赖。这样所有的依赖都被自动收集了。下次 `someRef` 变化时，就会触发该 effect 的调度器（重新执行 `fn`）。

## 总结与扩展

### 核心要点回顾

| 特性 | computed | watch | watchEffect |
|------|---------|-------|------------|
| 缓存 | ✅ 基于 dirty 标志位 | ❌ 无缓存 | ❌ 无缓存 |
| 懒执行 | ✅ 访问时才求值 | ❌ 立即收集依赖 | ❌ 立即执行 |
| 依赖追踪 | 自动（getter 内） | 显式指定 | 自动（执行时） |
| 副作用 | ❌ 不应有 | ✅ 副作用容器 | ✅ 副作用容器 |
| 新旧值 | ❌ 无 | ✅ 有 | ❌ 无 |
| DOM 时机 | 随渲染周期 | 可配置 flush | 可配置 flush |

### 深化理解：响应式系统的三个核心抽象

Vue 3 的响应式系统由三个抽象构成：

1. **`Proxy`（数据层）**：拦截对象的读取和写入操作，实现 `track()` 和 `trigger()`
2. **`Dep`（依赖层）**：每个响应式属性维护一个 `Set<ReactiveEffect>`，记录所有依赖它的 effect
3. **`ReactiveEffect`（副作用层）**：`computed`、`watch`、`watchEffect`、渲染器都是 `ReactiveEffect` 的不同配置形态

理解这三层的关系，就能真正吃透 Vue 3 的响应式原理：

```
        ┌───────────────────┐
        │  reactive/ref     │  ← 数据层（Proxy）
        │  dep: Set<Effect> │  ← 依赖收集
        └────────┬──────────┘
                 │ track / trigger
                 ▼
        ┌───────────────────┐
        │  ReactiveEffect   │  ← 副作用层
        │   ├ computed      │  ← scheduler 设 dirty
        │   ├ watch         │  ← scheduler 回调
        │   ├ watchEffect   │  ← scheduler 重新执行
        │   └ render        │  ← scheduler 执行 diff
        └───────────────────┘
```

### 进一步阅读

- Vue 3 源码：`packages/reactivity/src/computed.ts`
- Vue 3 源码：`packages/runtime-core/src/apiWatch.ts`
- Vue 3 响应式指南：[Reactivity in Depth](https://vuejs.org/guide/extras/reactivity-in-depth.html)
- Vue 3.4 `onWatcherCleanup` RFC：[Rationale and Design](https://github.com/vuejs/core/pull/9729)
- Evan You 关于 `computed` 缓存设计的演讲：[Vue.js London 2020](https://www.youtube.com/watch?v=9CUU7mYNU84)
