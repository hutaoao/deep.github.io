---
layout: post
title: "Vue 自定义指令与 3.4+ 编译器宏（defineModel / defineOptions / v-bind 同名简写）"
date: 2026-12-04 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [自定义指令, defineModel, script setup, 编译器宏, getSSRProps]
description: >
  面试向拆解 Vue 自定义指令：为什么它是唯一"直接操作 DOM"的复用方式、七个钩子与
  Vue 2 的迁移对应关系、实测的钩子调用顺序、binding 到底有哪些字段、script setup 里
  v 开头的变量为什么不用注册就能当指令用，以及 3.4/3.5 的编译器宏（defineModel /
  defineOptions / v-bind 同名简写）实际编译成了什么。
---

## 一句话概括

Vue 里复用代码有三种方式，面试常被要求"说清三者的分工"：

| 方式 | 复用的是什么 | 典型场景 |
|---|---|---|
| 组件 | 结构 + 逻辑 | 页面区块、UI 单元 |
| 组合式函数（composable） | **有状态的逻辑** | 请求封装、表单逻辑、防抖 |
| 自定义指令 | **底层 DOM 访问的逻辑** | 聚焦、点击外部关闭、曝光监听 |

官方对它的定位很克制，原话是：**"只有当所需功能只能通过直接的 DOM 操作来实现时，才应该使用自定义指令。"** 这句话是面试的得分点——它同时回答了两个追问："什么时候用"（指令天生碰 DOM）和"为什么不推荐滥用"（`v-bind` 这类内置指令更高效、对 SSR 更友好）。

而 3.3 到 3.5 这批版本，补的是另一半问题：**在 `<script setup>` 里少写样板代码**。`defineModel` 一个宏顶掉 `props.modelValue + emit('update:modelValue')` 那一对；`v-bind` 同名简写把 `:id="id"` 缩成 `:id`。这些都不是运行时 API，而是**编译器宏**——它们不处理数据流，只负责把你的代码"翻译"成原来的写法。

所以这篇分两半：**自定义指令（DOM 复用的老问题）+ 编译器宏（3.4/3.5 的新写法）**。文中的顺序、编译产物、警告文案都在 Vue 3.5.42 上实跑验证过。

## 核心知识点

### 1. 七个钩子与 Vue 2 的迁移对应

指令本质是**一个带生命周期钩子的普通对象**，钩子全部可选。Vue 2 → 3 做了一次改名，目的是让指令钩子和组件生命周期"名字对齐"：

| Vue 2 | Vue 3 | 调用时机 |
|---|---|---|
| `bind` | `beforeMount` | 元素挂载到 DOM 前 |
| `inserted` | `mounted` | 元素的父组件及其所有子节点挂载完成后 |
| — | **`created`（新）** | 元素的 attribute / 事件监听器被应用**之前** |
| — | **`beforeUpdate`（新）** | 父组件更新前 |
| `update` | **已删除** | 与 `updated` 重复，官方直接砍掉 |
| `componentUpdated` | `updated` | 父组件及其子节点都更新后 |
| — | **`beforeUnmount`（新）** | 卸载前 |
| `unbind` | `unmounted` | 卸载后 |

两个最常被追问的点：

- **`created` 是干什么的？** 它在 el 的 attribute、事件监听器应用之前跑，此时 el 还没"成型"。真正需要它的时候不多，典型用法是**在事件监听器绑上去之前先改点什么**（比如提前挂一个标识，或用 `getSSRProps` 配合做 SSR 兜底）。
- **为什么砍掉 `update`？** 因为它和 `updated` 的触发时机几乎完全重合，属于历史债。

**钩子和组件钩子的实际先后顺序**（实跑结论，很多八股文答错）：

```text
挂载：dir created → dir beforeMount → dir mounted → comp mounted
更新：dir beforeUpdate → dir updated → comp updated
app.unmount()：comp beforeUnmount → dir beforeUnmount → dir unmounted → comp unmounted
```

可以直接背的两条结论：

1. **元素上的指令 `mounted` 早于组件自己的 `mounted`**（两者都在 post 渲染队列里，指令的先入队）。
2. **卸载时组件的 `beforeUnmount` 最先跑**，之后才轮到指令的 `beforeUnmount` / `unmounted`。所以指令里清理 DOM 副作用（移除监听、断开 Observer）放在指令自己的钩子里是安全的，不会被组件抢在前头。

还有一个**必须记住的行为**：`beforeUpdate` / `updated` 是"父组件更新就触发"，**不是"值变了才触发"**。实测：只改了一个和指令无关的兄弟节点，指令的 `beforeUpdate` / `updated` 照样跑。所以指令里要判断"要不要干活"就得自己比 `oldValue`，不能假设"钩子被调用 = 值变了"。

### 2. binding 里到底有什么（实测字段）

钩子签名是 `(el, binding, vnode, prevVnode)`。`binding` 的字段实测为：

```text
arg, dir, instance, modifiers, oldValue, value
```

| 字段 | 含义 |
|---|---|
| `value` | 传给指令的值，`v-x="1 + 1"` 拿到的就是 `2`（表达式结果，不是字符串） |
| `oldValue` | 上一次的值，**只在 `beforeUpdate` / `updated` 里可用**（且"值没变"时也给你） |
| `arg` | 参数，`v-x:foo` → `"foo"` |
| `modifiers` | 修饰符对象，`v-x.foo.bar` → `{ foo: true, bar: true }` |
| `instance` | 使用该指令的组件实例（Vue 2 要绕 `vnode.context`，Vue 3 直接给） |
| `dir` | 指令的定义对象本身 |

两个容易踩的细节：

- **`vnode` 不在 `binding` 里**，它是第三个独立参数。答"binding 有哪些属性"时把它算进去就露馅了。
- **除 `el` 外其他参数都是只读的**（官方原话："不要更改它们"）。跨钩子共享数据官方推荐用元素的 `dataset`，实践中更常用 `el._xxx` 自定义属性（等价但不能被 `dataset` 序列化）。

⚠️ 一个非常隐蔽的坑（实跑验证）：**`binding` 是每次钩子调用时新建的快照对象**。如果你在 `mounted` 里写了个闭包捕获它，之后值更新了，闭包读到的还是旧值：

```js
const vClickOutside = {
  mounted(el, binding) {
    el._onClick = () => binding.value()   // ❌ 捕获的是 mounted 那一刻的 binding
    document.addEventListener('click', el._onClick)
  },
}
// 实测：父组件把回调换掉之后，点击触发的仍是旧回调（输出 v1, v1）
// ✅ 正确做法：把最新值存到 el 上，调用时现取
const vClickOutsideFixed = {
  mounted(el, binding) {
    el._cb = binding.value
    el._onClick = (e) => !el.contains(e.target) && el._cb(e)
    document.addEventListener('click', el._onClick)
  },
  updated(el, binding) {
    el._cb = binding.value           // 值更新时同步
  },
  beforeUnmount(el) {
    document.removeEventListener('click', el._onClick)  // 不清会漏内存
  },
}
```

### 3. 注册：三种方式，但 `<script setup>` 有"编译器魔法"

```js
// ① 全局注册：所有组件可用
app.directive('focus', { mounted: (el) => el.focus() })

// ② 组件局部（Options API，或普通 <script>）
export default {
  directives: { focus: { mounted: (el) => el.focus() } },
}

// ③ <script setup>：v 开头的驼峰变量，直接当局部指令用
const vFocus = { mounted: (el) => el.focus() }   // 模板里写 v-focus
```

第 ③ 种是面试的加分点，它**不是运行时约定，而是编译期识别的结果**。实跑 `@vue/compiler-sfc`，模板 `<input v-focus />` 编译出来是：

```js
// 编译产物（找得到同名绑定 → 直接内联，零运行时开销）
_withDirectives(_createElementVNode("input", null, null, 512 /* NEED_PATCH */), [
  [vFocus]
])

// 找不到同名绑定（比如模板里写了没声明的 v-other）→ 回退到运行时解析
const _directive_other = _resolveDirective("other")
```

所以结论是三句话：**命名必须是 `v` + 首字母大写的驼峰**（`vMyDirective` ↔ `v-my-directive`）；**编译器能找到就内联，找不到才走 `resolveDirective`**；**两者都没注册时会警告 `Failed to resolve directive: xxx`，指令静默不生效**（不报错，页面就是没效果，排查时别往别处想）。

### 4. 简化写法：函数简写 / 对象字面量 / 动态参数

```js
// ① 函数简写：只在 mounted 和 updated 做同一件事时用（实测两个时机都会被调用）
app.directive('color', (el, binding) => { el.style.color = binding.value })

// ② 对象字面量：一个指令传多个值（指令可以接收任何 JS 表达式）
// 模板：<div v-demo="{ color: 'white', text: 'hello' }"></div>
const vDemo = (el, binding) => {
  el.style.color = binding.value.color
  el.textContent = binding.value.text
}

// ③ 动态参数：v-example:[arg]="value"，arg 变化时 binding.arg 跟着变
```

### 5. 两个真正常用的业务指令

判断"该不该写成指令"的标准就一句话：**这件事离开 DOM 就做不成**。

```js
// 场景一：点击元素外部关闭弹窗/下拉（监听 document，判断事件目标是否在元素内）
const vClickOutside = {
  mounted(el, binding) {
    el._cb = binding.value
    el._onClick = (e) => { if (!el.contains(e.target)) el._cb(e) }
    document.addEventListener('click', el._onClick)
  },
  updated(el, binding) { el._cb = binding.value },
  beforeUnmount(el) { document.removeEventListener('click', el._onClick) },
}

// 场景二：曝光埋点 / 图片懒加载（IntersectionObserver 替代 scroll 监听）
const vExposure = {
  mounted(el, binding) {
    el._io = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        binding.value()          // 上报一次
        el._io.disconnect()      // 只报一次，顺手断开
      }
    })
    el._io.observe(el)
  },
  unmounted(el) { el._io?.disconnect() },
}
```

注意这两个例子的共同点：**都要在卸载钩子里清理**。定时器、全局事件监听、Observer、`AbortController`——这是自定义指令面试必然追问的一环。用 `IntersectionObserver` 而不是 `scroll` 监听的原因也值得带一句：scroll 回调每帧触发，且读 `getBoundingClientRect()` 会强制同步布局；Observer 是浏览器侧算好再通知你。

### 6. 用在组件上：单根能落，多根直接失效

- 指令写在**组件**上时，它会**始终应用到组件的根节点**（和透传 attribute 类似）。实测单根组件上，`el` 就是组件的根 DOM。
- 组件有**多个根节点**时，指令**被忽略**，并且 dev 环境会警告（实测原文）：

```text
[Vue warn]: Runtime directive used on component with non-element root node.
The directives will not function as intended.
```

- 官方态度是**不推荐**在组件上用自定义指令，而且有个硬限制：**指令不能像 attribute 那样通过 `v-bind="$attrs"` 转交给别的元素**。

### 7. SSR：只有 `getSSRProps` 会跑

因为指令大多在操作 DOM，而服务端没有 DOM，官方的原话是：**"因为大多数的自定义指令都包含了对 DOM 的直接操作，所以它们会在 SSR 时被忽略。"** 如果你希望某个指令在服务端也产出结果，可以加 `getSSRProps` 钩子——**它只接收 `binding` 一个参数**，返回值会被当作 attribute 渲染进 HTML：

```js
const vMark = {
  mounted(el, binding) { el.id = binding.value },
  getSSRProps(binding) { return { id: binding.value } },   // 只收 binding
}
```

实跑 SSR 验证：`created` / `beforeMount` / `mounted` **一次都没被调用**，只有 `getSSRProps` 执行，渲染结果是 `<div class="x" data-x="1">hello</div>`——即返回值直接变成了 HTML 属性，hydration 后客户端钩子才接手。面试问到"自定义指令在 SSR 里怎么办"，这就是标准答案。

### 8. 3.3 → 3.5 的编译器宏：它们编译成了什么

先给结论：**这些宏都不是运行时 API，不需要 `import`，也 import 不得**。实跑 `@vue/compiler-sfc`：写了 `import { defineModel } from 'vue'` 会得到编译期警告——

```text
[@vue/compiler-sfc] `defineModel` is a compiler macro and no longer needs to be imported.
```

| 宏 | 版本 | 编译成什么 |
|---|---|---|
| `defineProps` / `defineEmits` | 3.0 | props / emits 选项 + 类型推导 |
| `defineExpose` | 3.0 | 暴露给模板 ref 的实例属性 |
| `defineOptions` | 3.3+ | 提升到模块作用域的组件选项 |
| `defineSlots` | 3.3+ | 仅类型提示（无运行时产物） |
| `defineModel` | 3.3 实验 / **3.4 稳定** | 一个 prop + 一个 update 事件 + 一个可写 ref |
| props 解构保持响应式 | **3.5 稳定** | 访问处编译成 `props.xxx` |
| `v-bind` 同名简写 | **3.4** | 编译期展开成 `:id="id"` |

#### 8.1 `defineModel`：一个宏 = prop + 事件 + 可写 ref

```text
// 源码
const model = defineModel({ required: true })

// 编译产物（实跑 @vue/compiler-sfc，Vue 3.5.42）—— 三行摘要
props:  { modelValue: { ...{ required: true } }, modelModifiers: {} }
emits:  ['update:modelValue']
setup:  const model = _useModel(__props, 'modelValue')
```

要点：

- **写 `model.value = x` 就等于 `emit('update:modelValue', x)`**，父组件的 `v-model` 收到。3.5 的运行时实现是 `useModel`（3.3/3.4 早期是"本地 ref + watch + emit"的写法，产物不同，别背具体实现，记语义）。
- **命名**：`defineModel('count')` 声明的 prop 是 `count`、事件是 `update:count`，父组件用 `v-model:count`。
- **修饰符与转换**：解构第二个返回值拿 modifiers，用 `get` / `set` 做读写转换：

  ```js
  const [text, textModifiers] = defineModel({
    set(v) { return textModifiers.trim ? v.trim() : v },   // 同步回父组件前转换
  })
  ```
- **`default` 的坑**（官方 WARNING）：如果给了默认值而父组件根本没传 `v-model`，父子会**不同步**——子组件里是 `1`，父组件的 ref 还是 `undefined`。所以默认值只适合"父组件一定会传、只是可能传 `undefined`"的场景，否则老实用 `required` 或不用默认值。

#### 8.2 `v-bind` 同名简写：`<img :id :src />`

3.4 起，**attribute 和绑定值同名时可以省略值**。实跑编译产物：

```js
// 源码：<img :id :src="src" :alt />
// 编译产物里 attributes 部分（实跑 @vue/compiler-sfc）
const attributes = { id: _ctx.id, src: _ctx.src, alt: _ctx.alt }
```

注意它是**编译期按同名表达式展开**的，不是运行时魔法：所以作用域里没有同名变量时，编译不报错、运行时拿到 `undefined`（别指望它帮你兜底）。这个特性官方设计时的顾虑就是"容易和布尔 attribute 混淆"，但考虑到 `v-bind` 本来就是动态的，最终让它的行为"更像 JavaScript"。

#### 8.3 `defineOptions` 与 props 解构

```js
// defineOptions：把原来必须另开 <script> 块的选项写进 <script setup>
defineOptions({ name: 'MyInput', inheritAttrs: false })
```

**注意**：它会被**提升到模块作用域**，所以不能引用 `<script setup>` 里声明的局部变量。实跑确认：`defineOptions({ name: foo })` 编译不报错，但产物把 `{ name: foo }` 原样提到了模块作用域，运行时直接 `foo is not defined`——只能用字面量常量。

props 解构保持响应式在 **3.5 稳定**，`withDefaults` 可以不用了：

```ts
const { count = 0, msg = 'hi' } = defineProps<{ count?: number; msg?: string }>()
// 编译期把 count 的每次访问改写成 props.count，所以响应式没丢
// 但 watch(count) 会编译报错，必须写 watch(() => count)
```

最后一行是最容易被问倒的地方：**解构出来的变量本身不是 ref，只是"访问时被改写"的语法糖，所以凡是需要"传引用"的地方（watch 第一个参数、传给 composable）都得包一层 getter。**

## 其实你每天都在用

- 打开弹窗外点一下就关、下拉菜单失去焦点就收起，底层基本都是 `v-click-outside` 这类指令。
- 电商列表滚动到可视区才上报曝光、图片进入视口才加载，都是 `IntersectionObserver` 指令。
- 表单第一个输入框自动聚焦，官方文档就把它当作 `v-focus` 的教科书例子——它比原生 `autofocus` 强的地方是"动态插入的元素也生效"。
- Element Plus 的 `v-loading`、`v-infinite-scroll`，Ant Design Vue 的 `v-decorator`，全是同一种模式（指令 + 挂载时创建实例 + 卸载时销毁）。
- 组件库的弹窗 `<Dialog v-model:visible="show" />`，就是 `defineModel('visible')` 的标准用法。
- 写 `<img :src :alt />` 少敲了一半字符，这是 3.4 的 `v-bind` 同名简写。
- 以前为了在 `<script setup>` 里写 `name` / `inheritAttrs` 不得不另开一个 `<script>` 块，3.3 之后用 `defineOptions` 一行解决。
- Vue DevTools 里看到组件名是你文件名（`UserList.vue` → `UserList`），那是 `__name` 被自动注入了。

## 常见误解（FAQ）

**❌ 误区1："binding 里有 `vnode` 和 `prevVnode`"**

错。实测 `binding` 的字段只有 `arg / dir / instance / modifiers / oldValue / value`。`vnode` 和 `prevVnode` 是钩子的第 3、4 个参数。另外一个版本是"`binding` 里能拿到组件实例 `this`"——Vue 3 里是 `binding.instance`（Vue 2 才要绕 `vnode.context`）。

**❌ 误区2："`updated` 只在指令的值发生变化时才触发"**

错。`beforeUpdate` / `updated` 跟随**父组件的更新**触发，值没变也会跑（实测：改一个无关的兄弟节点，钩子照样执行）。想"只在值变了才干活"得自己对比 `oldValue`。顺带纠正另一半：`oldValue` 是"无论值是否更改都可用"，不是"只在变过之后才有值"。

**❌ 误区3："在 `<script setup>` 里用局部指令必须写 `directives` 选项"**

不需要。**任何 `v` + 驼峰的变量都会被编译器当作局部指令**，编译期直接内联进 `withDirectives`（实跑产物是 `[[vFocus]]`，连运行时解析都省了）。而 `<script setup>` 里也**没有** `directives` 这个配置项——写不了，也不用写。

**❌ 误区4："在组件上写自定义指令可以作用到子组件内部的元素"**

不行。指令写在组件上时**只会落到根节点**（和透传 attribute 一致）；根节点不是元素（多根、Fragment、文本、Teleport 根）时指令**被忽略**并警告 `Runtime directive used on component with non-element root node.`。而且和 attribute 不同，指令**无法通过 `v-bind="$attrs"` 转交**。所以官方直接写了"不推荐"。

**❌ 误区5："`mounted` 里拿到的 `binding` 会一直是最新的"**

错，而且很隐蔽。`binding` 是每次钩子调用**新建的快照**，闭包捕获它就会锁住当时的值（实测点击仍触发旧回调）。正确做法是把最新值存到 `el` 上（`mounted` 存一次、`updated` 同步），调用时现取。

**❌ 误区6："`defineModel` 要从 vue 里 import"**

不用，也不能。它是编译器宏，`import { defineModel } from 'vue'` 会得到编译期警告 `defineModel is a compiler macro and no longer needs to be imported.`——`defineProps` / `defineEmits` / `defineOptions` / `defineSlots` 同理。

**❌ 误区7："`defineOptions` 里能写 `defineOptions({ name: someVariable })`"**

不能。选项对象会被提升到模块作用域，引用 `setup` 内的局部变量会在运行时 `is not defined`（实测编译不报错，所以坑更深）。只能用字面量。

**❌ 误区8："自定义指令在 SSR 里完全不生效，用不了"**

准确说法是：**默认被忽略，但可以用 `getSSRProps` 补**。它只接收 `binding` 一个参数，返回值作为 attribute 渲染进服务端 HTML（实测 `created` / `beforeMount` / `mounted` 都不调用，只有它执行）。

## 一句话总结

**自定义指令是 Vue 三种复用方式里唯一"被允许碰 DOM"的那种——所以它的用法是"能不用就不用"，代价是 SSR 默认失效（要补 `getSSRProps`）、写在组件上只落根节点（多根直接忽略）、`binding` 是快照（闭包会锁旧值）；而 3.4/3.5 的编译器宏（`defineModel` / `defineOptions` / `v-bind` 同名简写）不是运行时能力，它们是"把老写法在编译期改写掉"的语法糖——记语义，别背实现。**
