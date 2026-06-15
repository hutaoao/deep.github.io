---
layout: post
title: "Composition API设计理念深度解析"
date: 2026-06-29
tags: [Vue3, Composition API, setup, 组合式]
categories: [前端核心, Vue]
---

## 一句话概括

Vue 3 的 Composition API 是一套基于函数组合的响应式逻辑组织方案，它通过 `setup` 函数作为组件入口，借助 `ref`、`reactive`、`computed`、`watch` 等底层响应式 API，将原本分散在 Options API 各选项中的逻辑按关注点聚合并自由组合，从根本上解决了大型组件中逻辑碎片化和复用困局。

## 背景与意义

### Options API 的辉煌与隐忧

Vue 2 时代的 Options API 以其"声明式"和"分选项"的直观设计赢得了大量开发者的青睐。将数据放在 `data`、方法放在 `methods`、计算属性放在 `computed`、侦听器放在 `watch`——这种按"类型"而非按"功能"划分代码的方式，在小规模组件中干净利落。

然而，随着前端应用日趋复杂，Options API 的局限性逐渐暴露：

**逻辑碎片化**。当一个组件需要处理多个不同的功能关注点（如用户认证、数据采集、实时推送、权限校验）时，每个关注点的代码被强行拆散到各个选项中。一个功能的 `data` 在顶部，`methods` 在中间，`watch` 在底部——阅读一个功能点需要在文件中反复上下跳转。这在超过 500 行的组件中尤为痛苦。

**Mixin 的致命缺陷**。Options API 时代，官方的逻辑复用方案是 mixin。但 mixin 存在几个根深蒂固的问题：
- **命名冲突**：多个 mixin 可以定义同名的 `data` 字段或 `methods` 方法，后加载的会静默覆盖前者
- **来源不透明**：组件模板中使用的变量，你无法快速判断是来自哪个 mixin，调试时需要逐个排查
- **隐式依赖**：mixin 之间可以相互依赖对方的属性，形成隐式耦合，重构风险极高

**TypeScript 支持不够友好**。Options API 中 `this` 的上下文是魔法注入的，TypeScript 难以推断 `this.methods`、`this.data` 等属性的类型，需要大量类型断言和装饰器辅助。

### Composition API 的设计目标

面对这些痛点，Vue 团队在设计 Composition API 时明确了几个核心目标：

1. **按关注点聚合逻辑** —— 让相关代码待在一起
2. **纯粹的函数级复用** —— 告别 mixin 的副作用，拥抱组合优于继承
3. **更好的 TypeScript 推断** —— 普通的函数调用和变量声明天然支持类型推导
4. **更小的生产体积** —— 按需导入的 API 让 tree-shaking 更高效
5. **渐进式的迁移路径** —— 与 Options API 共存，不强制全量重写

## 概念与定义

### 核心哲学："组合"而非"选项"

Composition API 的核心理念是：**组件不再是一个被动的选项容器，而是一个主动的函数执行体**。开发者的思维模式从"我有哪些选项需要填写"转变为"我的组件要完成什么功能"。

在 Options API 中，你是这样思考的：

```
我要写一个用户列表组件
→ data 里声明 users 数组、loading 状态、error 信息
→ methods 里写 fetchUsers 方法
→ mounted 里调用 fetchUsers
→ computed 里写 filteredUsers
```

在 Composition API 中，你的思维变成了：

```
我要实现"加载用户列表"这个功能
→ 声明响应式状态 users, loading, error
→ 定义 fetchUsers 函数
→ 在 onMounted 中调用
→ 导出 filteredUsers

然后再实现"搜索过滤"这个功能
→ ...
```

每个功能块是完整的、独立的、可提取的。

### setup 函数

`setup` 是 Composition API 的入口函数，它在组件实例创建之前被调用，是组件逻辑的"编排中心"。基本签名如下：

```typescript
interface ComponentOptions {
  setup?: (props: Props, context: SetupContext) => object | void
}

interface SetupContext {
  attrs: Record<string, any>
  slots: Slots
  emit: (event: string, ...args: any[]) => void
  expose: (exposed?: Record<string, any>) => void
}
```

`setup` 返回的对象中的属性会暴露到模板中，方法与 Options API 的 `data` 和 `methods` 类似。

### 核心响应式 API

**`ref`**：将任意值包装成响应式引用。适合声明基本类型值。

```typescript
import { ref } from 'vue'
const count = ref(0)          // Ref<number>
console.log(count.value)      // 0
count.value++                 // 响应式更新
```

**`reactive`**：将对象本身变成响应式。适合声明对象或数组。

```typescript
import { reactive } from 'vue'
const state = reactive({ count: 0, users: [] })
console.log(state.count)      // 0 —— 直接访问，无需 .value
```

**`readonly`**：创建响应式对象的只读代理。适合向子组件传递不可修改的 props。

```typescript
import { reactive, readonly } from 'vue'
const state = reactive({ count: 0 })
const readOnlyState = readonly(state)
readOnlyState.count++         // 静默失败，生产环境会 warning
```

### 生命周期钩子

Composition API 中的生命周期钩子以 `on` 前缀命名，一一对应 Options API 的钩子：

| Options API | Composition API |
|:---|:---|
| `beforeCreate` | —— 在 `setup` 内部直接编写 |
| `created` | —— 在 `setup` 内部直接编写 |
| `beforeMount` | `onBeforeMount` |
| `mounted` | `onMounted` |
| `beforeUpdate` | `onBeforeUpdate` |
| `updated` | `onUpdated` |
| `beforeUnmount` | `onBeforeUnmount` |
| `unmounted` | `onUnmounted` |
| `activated` | `onActivated` |
| `deactivated` | `onDeactivated` |
| `errorCaptured` | `onErrorCaptured` |
| `renderTracked` | `onRenderTracked` |
| `renderTriggered` | `onRenderTriggered` |

**⚠️ 注意**：`beforeCreate` 和 `created` 在 Composition API 中没有对应的 `onXxx`，因为 `setup` 本身就运行在 `beforeCreate` 和 `created` 之间，`setup` 函数内的代码相当于同时替代了这两个钩子。

## 最小示例

一个简单的计数器组件，分别用 Options API 和 Composition API 实现，感受差异。

**Options API 版本：**

```vue
<script>
export default {
  data() {
    return { count: 0, step: 1 }
  },
  computed: {
    double() { return this.count * 2 }
  },
  methods: {
    increment() { this.count += this.step },
    reset() { this.count = 0 }
  },
  watch: {
    count(newVal) {
      console.log(`count 变为 ${newVal}`)
    }
  }
}
</script>

<template>
  <div>
    <p>{{ count }} → 双倍: {{ double }}</p>
    <button @click="increment">+{{ step }}</button>
    <button @click="reset">重置</button>
  </div>
</template>
```

**Composition API 版本：**

```vue
<script setup>
import { ref, computed, watch } from 'vue'

const count = ref(0)
const step = ref(1)
const double = computed(() => count.value * 2)

function increment() { count.value += step.value }
function reset() { count.value = 0 }

watch(count, (newVal) => {
  console.log(`count 变为 ${newVal}`)
})
</script>

<template>
  <div>
    <p>{{ count }} → 双倍: {{ double }}</p>
    <button @click="increment">+{{ step }}</button>
    <button @click="reset">重置</button>
  </div>
</template>
```

注意到几个关键变化：
- 没有 `this`，全部是普通的变量和函数声明
- 逻辑紧密聚合在一起，`count`、`step`、`double`、`increment`、`reset`、`watch` 都在相邻行
- `<script setup>` 语法糖让代码更加简洁，导入即用，无需 export return

## 核心知识点拆解

### 一、setup 函数的执行时机与 this 指向

这是面试中最高频的考点之一。

**执行时机：**

```
组件实例创建过程：
1. 初始化 props
2. beforeCreate 钩子（已废弃 Vue 2 的 beforeCreate 概念，但 setup 在此时机运行）
3. 调用 setup(props, context)
4. created 钩子（setup 返回后即相当于 created 阶段完成）
5. 模板编译生成 render 函数
6. beforeMount
7. mounted
```

`setup` 运行在组件实例被完全创建之前。具体来说，它在 `beforeCreate` 之后、`created` 之前执行。这意味着在 `setup` 中，你可以访问 `props`，但不能访问 `this`。

**为什么 setup 中不能访问 this？**

"等一等，"你可能会问，"Vue 2 的 `data` 和 `methods` 不都是在实例化之前定义的吗？它们为什么能用 `this`？"

关键在于 **执行上下文** 的差异：

- Options API 中的 `data`、`methods`、`computed` 等方法被 Vue 内部包装后，绑定到组件实例上再执行，所以 `this` 指向组件实例
- `setup` 本身就是一个普通的函数调用，Vue 在调用 `setup` 时组件实例还没完全初始化好，`this` 此时还没有绑定到组件实例上

更深层的原因是 **设计理念的转变**：Composition API 想要消除对 `this` 的依赖，因为 `this` 是隐式的、不确定的、难以静态分析的。当所有东西都通过 `this.xxx` 访问时，TypeScript 的推断变得困难。而 `setup` 中的变量就是普通的局部变量，TypeScript 可以完美地推断类型。

```typescript
// ❌ 错误：setup 中不能访问 this
setup() {
  console.log(this)        // undefined（开发环境）或组件实例的代理（生产环境，但不建议依赖）
  console.log(this.count)  // undefined
}

// ✅ 正确：通过 props 参数访问传入的 props
setup(props) {
  console.log(props.count)
}
```

**实验验证：**

```vue
<script>
import { onMounted } from 'vue'

export default {
  data() { return { msg: 'hello' } },
  setup() {
    console.log('setup 执行')
    onMounted(() => {
      console.log('setup 中的 onMounted')
    })
  },
  beforeCreate() {
    console.log('beforeCreate')   // Vue 3 中不存在，仅在 Vue 2 兼容模式下
  },
  created() {
    console.log('created 执行, this.msg =', this.msg)
  },
  mounted() {
    console.log('mounted 执行')
  }
}
</script>
```

控制台输出顺序：

```
setup 执行
created 执行, this.msg = hello
(模板 render)
setup 中的 onMounted    ← 注意：虽然写在 setup 中，但注册到了对应的生命周期
mounted 执行
```

可以看到，`setup` 比 `created` 更早执行，但 `onMounted` 注册的回调仍然会在 `mounted` 时触发。这是因为 Vue 内部通过一个全局变量 `currentInstance` 来追踪当前正在初始化的组件实例，`onMounted` 会在 `setup` 执行期间将回调注册到该实例的对应生命周期队列中，而非立即执行。

### 二、生命周期钩子在 Composition API 中的实现原理

这是一个非常巧妙的设计。Vue 内部维护了一个"当前活跃的组件实例"（current instance），当 `setup` 函数执行时，这个实例被压入一个栈中。

```typescript
// Vue 源码简化示意
let currentInstance: ComponentInternalInstance | null = null

function setCurrentInstance(instance: ComponentInternalInstance | null) {
  currentInstance = instance
}

function onMounted(fn: () => void) {
  if (currentInstance) {
    currentInstance.mounted.push(fn)
  } else {
    // 开发环境 warning：生命周期钩子必须在 setup 中同步调用
  }
}

// 组件挂载流程
function mountComponent(component, container) {
  const instance = createComponentInstance(component)
  setCurrentInstance(instance)
  
  // 执行 setup
  const setupResult = component.setup(component.props, setupContext)
  
  // setup 执行完毕，清空 currentInstance
  setCurrentInstance(null)
  
  // ...
}
```

**关键规则**：生命周期钩子函数必须在 `setup` 的同步代码块中调用。如果在异步回调中调用 `onMounted`，此时 `currentInstance` 已经变为 `null`，注册会失败：

```typescript
// ❌ 错误：异步调用生命周期钩子
setup() {
  setTimeout(() => {
    onMounted(() => { /* 永远不会执行 */ })  // Warning: 没有活跃的组件实例
  }, 100)
}

// ✅ 正确：同步调用
setup() {
  onMounted(() => { /* 正常执行 */ })
}
```

这个"全局状态 + 栈"的设计模式在 React 的 Hooks 中也有类似实现（如 `useState`、`useEffect` 依赖 `Fiber` 链表）。Vue 的设计更简洁——因为它不需要维护调用顺序（Vue 的注册可以顺序无关），而 React Hooks 严格依赖调用顺序来匹配状态。

### 三、reactive / ref / readonly 深度对比

这三个是最基础的响应式 API，理解它们的区别和适用场景至关重要。

**ref 的实质**：

`ref` 本质上是一个语法糖。当你调用 `ref(0)` 时，内部创建了一个 `{ value: 0 }` 的对象，然后对这个对象调用 `reactive`：

```typescript
// 简化实现
function ref<T>(value: T): Ref<T> {
  return reactive({ value }) as Ref<T>
}
```

这就是为什么 `ref` 值必须通过 `.value` 访问——它只是对象上的一个属性。

**reactive 的限制**：

`reactive` 直接返回原始对象的 Proxy 代理，所以：
- 只能用于对象类型（Object, Array, Map, Set 等）
- 不能用于原始类型（number, string, boolean）
- 重新赋值会丢失响应式（因为新值是一个新对象，不再被代理）

```typescript
let state = reactive({ count: 0 })
// ❌ 丢失响应式
state = reactive({ count: 1 })  // 这实际上创建了一个新代理，但再也追踪不到了

// ✅ 使用 ref 解决
const state = ref({ count: 0 })
state.value = { count: 1 }  // 响应式依然存在
```

**readonly 的实现**：

`readonly` 创建一个 Proxy，拦截所有 `set` 操作并阻止修改。它通常与 `reactive` 配合使用，用于暴露一个状态的只读版本：

```typescript
function useUser() {
  const user = reactive({ name: 'Alice', age: 25 })
  
  // 内部可以修改
  function updateName(name: string) { user.name = name }
  
  // 向外暴露只读引用
  return { user: readonly(user), updateName }
}
```

**三者转换关系：**

```
原始值  ──ref──▶  Ref<T>（响应式引用）
对象值  ──reactive──▶  Proxy 代理（深层响应式）
任意响应式  ──readonly──▶  Proxy 代理（只读阻止修改）
```

### 四、逻辑复用机制：Mixin vs Composable

这是 Composition API 最核心的设计优势所在。

**Mixin 的问题**：

假设我们有一个 `useLogging` mixin 和一个 `useAuth` mixin：

```javascript
// mixin 示例 —— 问题重重
const useLogging = {
  data() { return { logCount: 0 } },
  methods: {
    log(msg) { console.log(msg); this.logCount++ }
  },
  mounted() { this.log('组件已挂载') }
}

const useAuth = {
  data() { return { logCount: 0 } },  // ❌ 命名冲突！覆盖了 useLogging 的 logCount
  methods: {
    login() { /* ... */ }
  }
}

// 组件使用
export default {
  mixins: [useLogging, useAuth],
  mounted() {
    this.log('自己的挂载逻辑')  // 这会覆盖还是叠加？答案是叠加，但顺序依赖规则
  }
}
```

mixin 的问题集中体现为：**（1）命名空间污染**、（2）**来源不透明**、（3）**隐式依赖**。

**Composable 的解决方案**：

```typescript
// composable —— 清晰的来源、无冲突、完全组合
function useLogging() {
  const logCount = ref(0)
  
  function log(msg: string) {
    console.log(msg)
    logCount.value++
  }
  
  onMounted(() => log('组件已挂载'))
  
  return { logCount, log }
}

function useAuth() {
  const user = ref<User | null>(null)
  
  async function login(username: string, password: string) {
    user.value = await api.login(username, password)
  }
  
  return { user, login }
}

// 组件中使用 —— 每个变量都有明确来源
const { logCount, log } = useLogging()
const { user, login } = useAuth()
// 👆 一眼就知道 logCount 来自 useLogging，user 来自 useAuth
// 即使两者返回同名字段，通过解构重命名即可解决
const { logCount: authLogCount } = useAuth()  // ✅ 轻松解决命名冲突
```

**Composable 的优势总结：**

| 维度 | Mixin | Composable |
|:---|:---|:---|
| 命名冲突 | 静默覆盖，不可控 | 解构可重命名，零冲突 |
| 来源追踪 | 难以判断成员来自哪个 mixin | 变量通过解构获取，来源清晰 |
| 隐式依赖 | mixin 可依赖其他 mixin 的属性 | 参数显式传入，无隐式耦合 |
| 类型推断 | 通过 `this` 访问，TypeScript 难以推断 | 普通变量和函数，天然类型安全 |
| tree-shaking | 所有属性全量注入 | 解构即按需导入，可 tree-shaking |
| 测试 | 需要组件挂载环境 | 纯函数，单元测试零成本 |

## 实战案例

### 案例：一个具备搜索、分页和实时更新的用户列表组件

下面通过一个真实场景展示 Composable 如何按功能组织逻辑。

**核心 Composable：**

```typescript
// composables/useUserList.ts
import { ref, reactive, computed, watch, onMounted, onUnmounted } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

interface PaginationState {
  page: number
  pageSize: number
  total: number
}

export function useUserList() {
  // ===== 数据状态 =====
  const users = ref<User[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)

  // ===== 分页 =====
  const pagination = reactive<PaginationState>({
    page: 1,
    pageSize: 20,
    total: 0
  })
  const totalPages = computed(() => Math.ceil(pagination.total / pagination.pageSize))

  // ===== 搜索 =====
  const searchQuery = ref('')
  const debouncedQuery = ref('')

  // 搜索防抖
  let debounceTimer: ReturnType<typeof setTimeout>
  watch(searchQuery, (val) => {
    clearTimeout(debounceTimer)
    debounceTimer = setTimeout(() => {
      debouncedQuery.value = val
      pagination.page = 1  // 搜索时重置到第一页
    }, 300)
  })

  // ===== 数据加载 =====
  async function fetchUsers() {
    loading.value = true
    error.value = null

    try {
      const response = await fetch(
        `/api/users?page=${pagination.page}&pageSize=${pagination.pageSize}&q=${debouncedQuery.value}`
      )
      const data = await response.json()
      users.value = data.items
      pagination.total = data.total
    } catch (e: any) {
      error.value = e.message
    } finally {
      loading.value = false
    }
  }

  // 监听搜索和分页变化，自动重新加载
  watch([debouncedQuery, () => pagination.page, () => pagination.pageSize], fetchUsers)

  // ===== 实时轮询 =====
  let pollInterval: ReturnType<typeof setInterval> | null = null

  function startPolling(intervalMs = 30000) {
    stopPolling()
    pollInterval = setInterval(fetchUsers, intervalMs)
  }

  function stopPolling() {
    if (pollInterval) {
      clearInterval(pollInterval)
      pollInterval = null
    }
  }

  onMounted(() => {
    fetchUsers()
  })

  onUnmounted(() => {
    stopPolling()
  })

  // ===== 分页工具函数 =====
  function goToPage(page: number) {
    if (page >= 1 && page <= totalPages.value) {
      pagination.page = page
    }
  }

  function nextPage() { goToPage(pagination.page + 1) }
  function prevPage() { goToPage(pagination.page - 1) }

  return {
    users, loading, error,
    pagination, totalPages,
    searchQuery, debouncedQuery,
    fetchUsers,
    goToPage, nextPage, prevPage,
    startPolling, stopPolling
  }
}
```

**组件中使用：**

```vue
<script setup lang="ts">
import { useUserList } from '@/composables/useUserList'

const {
  users, loading, error,
  pagination, totalPages,
  searchQuery,
  goToPage, nextPage, prevPage,
  startPolling, stopPolling
} = useUserList()

// 开启实时刷新
startPolling(15000)
</script>

<template>
  <div class="user-list">
    <!-- 搜索框 -->
    <input v-model="searchQuery" placeholder="搜索用户..." />

    <!-- 加载中 -->
    <div v-if="loading">加载中...</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- 用户列表 -->
    <ul v-else>
      <li v-for="user in users" :key="user.id">
        {{ user.name }} - {{ user.email }}
      </li>
    </ul>

    <!-- 分页 -->
    <div class="pagination">
      <button :disabled="pagination.page <= 1" @click="prevPage">上一页</button>
      <span>第 {{ pagination.page }} / {{ totalPages }} 页（共 {{ pagination.total }} 条）</span>
      <button :disabled="pagination.page >= totalPages" @click="nextPage">下一页</button>
    </div>
  </div>
</template>
```

**这段代码的亮点**：
- `useUserList` 是一个独立的函数，可以在任何组件中复用
- 搜索、分页、自动加载、轮询逻辑凝聚在一个函数中，一目了然
- 测试时只需导入 `useUserList`，无需挂载组件

## 底层原理

### 一、setup 的返回到模板的桥接

当 `setup` 返回一个对象时，Vue 会将返回的属性挂载到组件渲染上下文中。这个过程在 Vue 源码中大致如下：

```typescript
// Vue 3 源码简化（runtime-core/src/componentRenderUtils.ts）
function setupRenderEffect(instance, initialVNode, container) {
  const { render, setupState } = instance

  instance.renderProxy = new Proxy(instance, {
    get(target, key) {
      // 访问优先级：setup 返回的 > props > data > computed > methods
      if (key in target.setupState) {
        return target.setupState[key]
      }
      if (key in target.props) {
        return target.props[key]
      }
      // ... 其他选项
    }
  })
}
```

模板中的 `{{ count }}` 会被编译为 `$setup.count`（在 `<script setup>` 模式下）或通过渲染代理查找（非 `<script setup>` 模式）。

### 二、响应式系统的核心：Proxy

Vue 3 的响应式系统基于 ECMAScript 的 `Proxy` API，这与 Vue 2 的 `Object.defineProperty` 有本质区别：

```typescript
// Vue 2 的 defineProperty —— 需要提前声明所有属性
// 无法检测属性新增和删除，需要 Vue.set / Vue.delete

// Vue 3 的 Proxy —— 拦截所有操作
function reactive(target: object) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key)           // 收集依赖
      return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver)
      trigger(target, key)         // 触发更新
      return result
    },
    deleteProperty(target, key) {
      const result = Reflect.deleteProperty(target, key)
      trigger(target, key)
      return result
    }
  })
}
```

Proxy 可以拦截几乎所有对象操作（get、set、deleteProperty、has、ownKeys 等），这也是 Vue 3 能支持 Map、Set 等原生集合类型的原因。

### 三、ref 的 .value 背后

`ref` 的 `.value` 实际上也是 Proxy 拦截的。`Ref` 类型的内部实现：

```typescript
class RefImpl<T> {
  private _value: T

  constructor(value: T) {
    this._value = toReactive(value)  // 如果 value 是对象，转为 reactive
  }

  get value() {
    track(this, 'value')    // 收集依赖——让 .value 成为响应式的关键
    return this._value
  }

  set value(newVal) {
    this._value = toReactive(newVal)
    trigger(this, 'value')  // 触发更新——赋值时通知所有依赖重新计算
  }
}

function ref<T>(value: T): Ref<T> {
  if (isRef(value)) return value
  return new RefImpl(value)
}
```

所以 `ref(0)` 的 `.value` 本质上是一个 getter/setter，在 get 时收集依赖，在 set 时触发更新。

### 四、生命周期注册机制

前面提到的 `currentInstance` 全局变量和生命周期注册机制，其核心实现在 `runtime-core` 中：

```typescript
// packages/runtime-core/src/apiLifecycle.ts
export function onMounted(fn: () => void, target?: ComponentInternalInstance | null) {
  const instance = target || getCurrentInstance()
  if (instance) {
    const lifecycleHooks = instance.mounted || (instance.mounted = [])
    lifecycleHooks.push(fn)
  }
}

// 执行时机 —— 在组件挂载完成后
function mountComponent(instance, container) {
  // ... 创建和挂载
  if (instance.mounted) {
    instance.mounted.forEach(fn => fn())
  }
}
```

这个"全局 tracker"的设计也被 React Hooks 借鉴（React 的 `currentlyRenderingFiber`），但两者的实现细节不同。Vue 不需要关心 Hooks 的调用顺序（因为生命周期钩子是"注册后等待触发"模式），而 React 需要严格依赖顺序来匹配 state（因为 `useState` 返回的是当前 Fiber 节点的状态）。

## 高频面试题解析

### Q1：setup 和 created 谁先执行？setup 中能访问 this 吗？

`setup` 先于 `created` 执行。`setup` 调用时组件实例尚未完全初始化，因此**不能访问 `this`**。如果要使用响应式数据，需要依赖 `ref`、`reactive` 等 API 返回值。`props` 通过参数传入可直接访问。

### Q2：为什么 Vue 3 不直接保留 Options API，而是引入 Composition API？

这是 Vue 3 最受争议的设计决策。核心原因有三：

1. **逻辑组织**：Options API 按选项类型分割代码，当组件功能增多时产生"横切关注点"问题——一个功能（如 websocket 连接）的 data、methods、created、destroyed 分散在文件各处。Composition API 以功能为单位组织代码，在复杂场景下可读性和可维护性大幅提升。

2. **逻辑复用**：mixin 的缺陷（命名冲突、来源不透明、隐式依赖）无法在 Options API 的框架内优雅解决。Composition API 的 composable 是纯函数组合，天然解决了这些问题。

3. **类型安全**：Options API 中通过 `this` 暴露属性和方法，TypeScript 的 `ThisType` 虽然可以模拟，但深度有限。Composition API 的变量和方法就是普通的 JavaScript 变量和函数，TypeScript 可以完美推断。

### Q3：ref 和 reactive 怎么选？什么场景用哪个？

经验法则：

| 场景 | 推荐 |
|:---|:---|
| 基本类型值（string, number, boolean） | `ref` |
| 需要整体替换的对象 | `ref` |
| 需要深层响应式且频繁读写属性的对象 | `reactive` |
| 从外部传入且不确定类型的值 | `ref` |
| 需要在多个 composable 之间传递的状态 | `ref` |
| 大型的静态数据结构 | `reactive`（省去 .value 的开销） |

一个常用模式：用 `ref` 包装"可替换的"值，用 `reactive` 包装"持久的结构"。

### Q4：Composition API 能否完全替代 Options API？什么时候应该用 Options API？

**可以共存，但不应轻易混用同一个组件**。在同一个组件中混用两者虽然技术上可行，但会造成逻辑分散，失去 Composition API 的核心优势。

**推荐使用 Composition API 的场景**：
- 组件逻辑较复杂（超过 200 行）
- 组件需要跨功能点复用逻辑
- 项目中大量使用 TypeScript
- 团队熟悉函数式编程模式

**仍然适合 Options API 的场景**：
- 极简的纯展示组件（几十行代码）
- 团队大部分成员尚不熟悉 Composition API
- 维护中的老旧项目，逐步迁移策略
- 需要 `provide / inject` 跨层级传递少量属性

Vue 官方推荐：新项目默认使用 Composition API（尤其 `<script setup>`），Options API 作为特殊场景的备选。

### Q5：onMounted 写在 setup 中，为什么能在 mounted 阶段触发？原理是什么？

这是通过"当前活跃实例"（currentInstance）机制实现的。当 Vue 创建组件实例并调用 `setup` 时，会先将该实例设置为全局的 `currentInstance`，然后执行 `setup` 函数。在 `setup` 同步代码中调用 `onMounted(fn)` 时，`fn` 被注册到 `currentInstance.mounted` 数组中。`setup` 执行完毕后，Vue 清空 `currentInstance`。等到组件的真实 `mounted` 钩子触发时，Vue 遍历执行之前注册的所有函数。

这就是为什么生命周期钩子**必须在 `setup` 的同步上下文**中调用——只有在 `setup` 执行期间，`currentInstance` 才指向正确的组件实例。

### Q6：什么是 `<script setup>`？它和普通的 `setup()` 函数有什么不同？

`<script setup>` 是 Vue 3.2+ 引入的编译时语法糖，本质上是将 `<script setup>` 块编译为普通的 `setup` 函数。

**差异对比**：

```vue
<!-- 普通 setup -->
<script>
import { ref, onMounted } from 'vue'
export default {
  setup() {
    const count = ref(0)
    onMounted(() => console.log('mounted'))
    return { count }  // 必须显式 return
  }
}
</script>

<!-- <script setup> -->
<script setup>
import { ref, onMounted } from 'vue'
const count = ref(0)
onMounted(() => console.log('mounted'))
// 不需要 return，所有顶层声明自动暴露给模板
</script>
```

**核心优势**：
1. 更少的样板代码（无需 `export default` 和 `return`）
2. 更好的 IDE 支持（自动完成和类型推断）
3. 编译时优化（静态分析，减少运行时开销）
4. 直接使用 `import` 的组件，无需 `components` 选项注册

## 总结与扩展

### Composition API 的设计哲学总结

Composition API 不是一个"新语法"，而是一套**全新的组织思维**。它的核心信条是：

> **把你的组件想象成一个个函数的组合，而不是一个个选项的集合。**

这种思维转变带来了几个深远的影响：

1. **代码组织从"按类型"变为"按功能"** —— 相关代码紧邻，无关代码自然分离
2. **逻辑复用从"继承混合"变为"函数组合"** —— 清晰、类型安全、无副作用
3. **组件定义从"声明式对象"变为"函数式表达式"** —— 更接近原生 JavaScript，减少框架魔法的神秘感
4. **状态管理从"实例属性"变为"普通变量"** —— 消除 `this` 的魔法，让代码可静态分析

### 与 React Hooks 的对比思考

Vue 的 Composition API 与 React Hooks 在"函数组合"的理念上是相通的，但实现细节差异巨大：

| 维度 | Vue Composition API | React Hooks |
|:---|:---|:---|
| 响应式机制 | Proxy 代理，自动追踪依赖 | 不可变数据 + 显式 setState |
| 重复渲染 | 仅追踪依赖变化，精准更新 | 每次 setState 全组件重新执行 |
| 调用顺序 | 不依赖顺序，可条件调用 | 严格依赖调用顺序 |
| 内存管理 | 组件实例持久化，hooks 注册后常驻 | 每次渲染重建，闭包捕获问题 |
| 学习曲线 | 新增 API 需学习，但规则较少 | Hooks 规则多（Concurrent Mode 进一步增加复杂度） |

### 未来趋势

Vue 生态系统正在围绕 Composition API 全面重构：

- **Pinia**（Vue 官方推荐的状态管理库）完全基于 Composition API 设计
- **Vue Router 4** 提供了 `useRouter`、`useRoute` 等 composable 接口
- 越来越多的第三方库提供 composable API（如 `@vueuse/core` 提供了 200+ 开箱即用的 composable）
- **编译时优化**：Vue 3.4+ 的响应式语法糖进一步减少样板代码，`ref.value` 可以被编译器自动解包

Composition API 不是 Vue 的终点，而是 Vue 迈向更工程化、更类型安全、更可组合的未来的起点。对于每一位 Vue 开发者而言，理解 Composition API 的设计理念，不仅是掌握一套新 API，更是获得一种更强大的**抽象与组合能力**——这种能力在任何前端框架中都是通用的。

---

*本文发布于 2026-06-29，基于 Vue 3.5+ 版本撰写。Vue 生态仍在快速发展，建议参考[官方文档](https://vuejs.org/guide/extras/composition-api-faq)获取最新信息。*
