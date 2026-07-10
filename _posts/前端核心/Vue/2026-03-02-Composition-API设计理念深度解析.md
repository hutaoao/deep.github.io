---
layout: post
title: "Composition API设计理念"
date: 2026-03-02
tags: [Vue3, Composition API, setup, 组合式]
categories: [前端核心, Vue]
---

## 一句话概括

Composition API 是 Vue 3 用**函数组合代替选项分散**的逻辑组织方案——把同一功能的数据、方法、侦听器写在一起，而不是散落在 `data`/`methods`/`watch` 各处，解决了 Options API 时代组件大了之后「一个功能改三处」的痛点。

## 核心知识点

### 1. Options API 的「逻辑碎片」问题

Options API 按**类型**分块（data 归 data、methods 归 methods），小组件清晰，超 300 行后灾难——一个「搜索」功能的数据在第 5 行、方法在第 150 行、watch 在第 400 行，改一个需求要在文件里跳三次。

```javascript
// Options API：三个关注点被迫混合
export default {
  data() { return { query: '', results: [], loading: false, page: 1 } },
  methods: { search() {/*...*/}, loadMore() {/*...*/}, resetFilters() {/*...*/} },
  watch: { query() {/*...*/}, page() {/*...*/} },
  mounted() { this.search() }
}
```

### 2. setup 作为逻辑入口

`setup` 在组件实例创建前、props 解析后执行，是 Composition API 的「容器」。在 `setup` 里你可以把同一关注点的所有逻辑（ref、computed、watch、生命周期钩子）写在一个函数里。

```typescript
import { ref, computed, watch } from 'vue'

// 把「搜索」这个关注点封装成一个函数
function useSearch() {
  const query = ref('')
  const results = ref([])
  const loading = ref(false)

  const search = async () => {
    loading.value = true
    results.value = await fetch(`/api?q=${query.value}`)
    loading.value = false
  }

  watch(query, search)

  return { query, results, loading, search }
}

// 组件里：
const { query, results, loading, search } = useSearch()
```

### 3. ref vs reactive 怎么选

| | `ref` | `reactive` |
|------|------|------|
| 基本类型 | ✅ 包装成对象 | ❌ 无法代理原始值 |
| 对象 | ✅ `.value` 访问 | ✅ 直接 `.xxx` |
| 解构 | ❌ 丢失响应式（需 `.value`） | ❌ 丢失响应式（需 `toRefs`） |
| 重新赋值 | ✅ `ref.value = ...` 保持响应式 | ❌ `obj = {...}` 断开响应式 |

**推荐**：统一用 `ref`。`.value` 虽然多三个字符，但规避了 `reactive` 解构丢失响应式的坑，逻辑函数返回的 `ref` 可以安全解构。

```typescript
// ❌ reactive 解构丢失响应式
const state = reactive({ count: 0 })
const { count } = state   // count 现在是普通数字，不再响应式

// ✅ 用 ref 或 toRefs
const count = ref(0)      // 直接 ref
const { count } = toRefs(reactive({ count: 0 }))  // 或用 toRefs
```

### 4. composable 的逻辑复用

Composition API 的终极威力在于**函数级复用**：把任意组合式逻辑抽成 `useXxx` 函数，任何组件都可以 `import` 使用，没有 Mixin 的命名冲突和来源不透明。

```typescript
// composables/useMouse.ts —— 任何组件都能复用
export function useMouse() {
  const x = ref(0)
  const y = ref(0)
  onMounted(() => window.addEventListener('mousemove', (e) => {
    x.value = e.pageX
    y.value = e.pageY
  }))
  return { x, y }
}

// 组件A
const { x, y } = useMouse()
// 组件B 也能用，互不干扰
const { x, y } = useMouse()
```

### 5. Composition API vs Options API 怎么选

- **Composition API**：组件 > 200 行、多关注点、有复用的 composable、需要 TypeScript 完善类型推导
- **Options API**：简单展示组件（< 100 行）、团队刚从 Vue 2 迁移
- 两者**可以共存**，Vue 3 不强制二选一。但官方文档和生态库（Pinia、VueUse）已经全面转向 Composition API。

## 「其实你每天都在用」

1. **封装 API 请求逻辑**：`useRequest` 组合 loading/error/data 三态，省去每个组件里手动管理这三件事。
2. **表单校验**：`useForm` 组合字段值、校验规则、错误信息，比散在 `data` + `methods` + `watch` 干净十倍。
3. **窗口尺寸 / 鼠标位置**：`useWindowSize`、`useMouse` 这类工具 composable 接上就能用，来自 VueUse 生态。
4. **暗色模式切换**：`useDark` 组合 `ref` + `watchEffect` + 操作 `document.documentElement.classList`，一行引入搞定。
5. **pinia store**：`const store = useUserStore()` —— Pinia 本身就是靠 Composition API 思想设计的，store 就是一个全局 composable。

## 常见误解（FAQ）

**❌ 误区：Composition API 是 Vue 3 独有的，Vue 2 用不了。** `@vue/composition-api` 插件让 Vue 2.6+ 也能用。很多团队在 Vue 2 项目里就已经提前用上 Composition API 为迁移做准备了。

**❌ 误区：setup 里不能用生命周期钩子。** `onMounted`、`onUnmounted` 等都可在 setup 里直接用，和 `mounted` / `destroyed` 选项一一对应。只是换了个写法，能力完全一样。

**❌ 误区：ref 解构会丢失响应式。** `ref` 解构**不会**丢失响应式——解构后拿到的还是同一个 ref 对象（引用类型），`count.value` 依然响应式。丢失响应式的是 `reactive` 解构（拿到的是原始值的拷贝）。

**❌ 误区：Composition API 比 Options API 「高级」。** 不是高级/低级的问题，是场景适配。30 行的展示组件用 Options API 完全够用；300 行有 5 个关注点的组件，Composition API 的可读性优势是碾压级的。

## 一句话总结

Composition API 不是 Vue 在耍新花样——它是把「按关注点组织代码」从设计模式变成了框架原语，让大型组件的可维护性不再靠开发者的「自觉」而是靠 API 的「强制」。
