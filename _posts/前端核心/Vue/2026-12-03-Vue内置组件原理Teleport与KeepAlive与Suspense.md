---
layout: post
title: "Vue 内置组件原理：Teleport / KeepAlive / Suspense"
date: 2026-12-03 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [Teleport, KeepAlive, Suspense, LRU缓存, activated, 内置组件]
description: >
  面试向拆解 Vue 三个内置组件：为什么它们必须是"内置"（渲染器特判而非普通组件）、
  Teleport 怎么做到 DOM 换位置但组件树关系不变、KeepAlive 缓存的是什么、
  key 到底是 name 还是组件本身、LRU 淘汰的实现细节与"淘汰不触发 deactivated"这个坑、
  以及 Suspense 的实验性状态与异步依赖收集机制。
---

## 一句话概括

`<Teleport>`、`<KeepAlive>`、`<Suspense>` 这三个组件有一个共同点，也是面试最想听的答案：

**它们都不是"普通组件"，而是渲染器里的特判分支。** 每个都带一个内部标记——`__isTeleport`、`__isKeepAlive`、`__isSuspense`——渲染器在 patch 时看到这些标记，就**不走普通的挂载/更新流程**，而是调用各自专用的一套 `process / hydrate / move / remove`。

vnode 上用 shapeFlag 做识别（实测 SharedFlags 值）：

| shapeFlag | 值 | 说明 |
|---|---|---|
| TELEPORT | 64 | 标在 `<Teleport>` 自己的 vnode 上 |
| SUSPENSE | 128 | 标在 `<Suspense>` 自己的 vnode 上 |
| COMPONENT_SHOULD_KEEP_ALIVE | 256 | **KeepAlive 给子 vnode 打的**：卸载时别真卸 |
| COMPONENT_KEPT_ALIVE | 512 | **KeepAlive 给子 vnode 打的**：挂载时复用旧实例 |

注意 KeepAlive 的特点：它自己不需要 shapeFlag 特判，而是**给子 vnode 打标记**来"遥控"渲染器的卸载/挂载行为（下面 2.2 会讲）。

为什么要这样设计？因为这三个组件要干的事，普通组件权限不够：

- **Teleport** 要决定"我这段 DOM 挂到哪个父节点下面"——普通组件的挂载容器是渲染器传进来的，改不了；
- **KeepAlive** 要决定"子组件卸载时别真卸，移到隐藏容器里存着"——这是卸载流程的内部逻辑；
- **Suspense** 要拦住子树渲染、等异步依赖全部 resolve——这要求它能在子组件 setup 抛异步时"挂起"整棵子树。

**一句话记忆：普通组件管"渲染什么"，内置组件管"什么时候渲染、渲染到哪、卸载时说声别卸"。** 这三件事只有渲染器说了算。

## 核心知识点

### 1. Teleport：DOM 换位置，组件树不变

面试官的经典问法：**"Teleport 的内容真的脱离父组件了吗？"**

**没有。脱离的只有 DOM 位置。** 逻辑上它依然是父组件的子节点——**组件树和 DOM 树在这里第一次分家**：

| 维度 | 结果 |
|---|---|
| DOM 挂载位置 | ✅ **变了**，挂到 `to` 指定的容器 |
| 组件树父子关系 | ✅ 不变（DevTools 里仍显示在原父组件下） |
| props / emit | ✅ 正常 |
| provide / inject | ✅ 正常（按组件树的 `parent` 链查找，不按 DOM 树） |
| 响应式、ref | ✅ 正常（`ref` 拿到的仍是真实 DOM） |
| CSS 继承 / 事件冒泡顺序 | ❌ **按真实 DOM 树走**——这是"分家"带来的副作用 |

这正是它比"手动 `appendChild` 挪 DOM"高明的地方——**模态框、全屏遮罩、tooltip 常要挂到 `body` 上躲开 `overflow: hidden` / `transform` 造成的层级问题，但又不想因此丢掉组件上下文。**

实现上，Teleport 会在**原位置留下两个注释节点**做占位（实测产物）：

```html
<!-- Teleport 渲染后，原位置留下占位符，真正内容被移到 #target -->
<div id="wrapper"><!--teleport start--><!--teleport end--></div>
<div id="target"><span id="inner">teleported</span></div>
```

这两个注释就是"锚点"（mainAnchor / targetAnchor），后续的移动、删除、hydration 全靠它们定位。

三个 props 的实际行为（都实测验证过）：

```vue
<template>
  <!-- to：CSS 选择器字符串，或直接传一个 DOM 元素 ref -->
  <Teleport to="#modal-root">...</Teleport>

  <!-- disabled：临时禁用传送，内容回到原位渲染（用于移动端弹窗改内联的场景） -->
  <Teleport to="#modal-root" :disabled="isMobile">...</Teleport>

  <!-- defer（Vue 3.5+）：目标元素由"同一轮渲染中靠后的组件"渲染时，等它出来再挂 -->
  <Teleport to="#late" defer>...</Teleport>
  <div id="late">我是后面才渲染出来的目标</div>
</template>
```

几个必须知道的细节：

- **动态切换 `to` 或 `disabled` 时，是移动 DOM，不是销毁重建。** 实测：从 `#target` 切到 `#target2`，`#target` 瞬间变空、同一个 DOM 节点出现在 `#target2` 里。**内容是被"搬"过去的，不是重新渲染的**，所以子组件实例、响应式状态（包括输入框里还没提交的值、滚动位置）都不会丢。面试官问"切来切去会不会丢状态"，答案是**不会**。
- **目标不存在会警告并渲染不出来**：`[Vue warn]: Failed to locate Teleport target with selector "#nope"`。原因是 Teleport 在**挂载阶段**就要 `resolveTarget`，如果目标是被同一个组件自己渲染的，那一轮它还不存在。所以官方建议目标放在整个 Vue 组件树之外，或者用 3.5 的 `defer`（源码里走 `queuePendingMount`，等渲染后钩子再挂）。
- **SSR 时要手动处理**：服务端渲染时 Teleport 的内容被收集到 `SSRContext.__teleportBuffers[target]`，不会直接出现在 HTML 流的对应位置，需要模板里手动把 buffer 插到目标元素中。这是 SSR 项目用 Teleport 最容易踩的一脚。

### 2. KeepAlive：缓存的是 vnode，key 不是 name

`<KeepAlive>` 是个**抽象组件**——它自己不产生任何 DOM，只负责管子组件的生死。

#### 2.1 数据结构：一个 Map + 一个 Set

```js
// 简化自 runtime-core/src/components/KeepAlive.ts
const cache = new Map()   // key → 缓存的 vnode
const keys = new Set()    // 只存 key，维护"访问顺序"，LRU 靠它
```

**第一个高频错答点来了**：cache 的 key 是什么？

```js
// 源码原文（简化）
const key = vnode.key == null ? comp : vnode.key
//                              ↑ 组件定义对象本身，不是组件 name！
```

所以 key 是 **`vnode.key`（你写的 `:key`）**，没写 `:key` 时退化成**组件定义对象（`vnode.type`）**——**不是组件 name**。`name` 只用于 `include` / `exclude` 的匹配，两者完全不是一回事。很多八股文写"key 由 name 或 key prop 决定"，前半句是错的。

**缓存的是什么？** `Map` 的 value 是 **vnode**，而 vnode 上面挂着 `vnode.component`（组件实例）和 `vnode.el`（真实 DOM）。所以准确说法是：**缓存 vnode，顺带把组件实例和真实 DOM 一起留住了。** 连带留住的还有：setup 里的响应式状态、watcher、后代的组件实例、DOM 状态（滚动位置、输入框内容）。

#### 2.2 命中 / 未命中，就两段代码

```js
// 未命中：新组件
keys.add(key)
if (max && keys.size > parseInt(max, 10)) {
  pruneCacheEntry(keys.values().next().value)   // 淘汰最久未用的（Set 头部）
}
vnode.shapeFlag |= ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE   // 256

// 命中：复用旧实例
vnode.el = cachedVNode.el                 // 复用真实 DOM
vnode.component = cachedVNode.component   // 复用组件实例（不重新 setup）
vnode.shapeFlag |= ShapeFlags.COMPONENT_KEPT_ALIVE          // 512
keys.delete(key); keys.add(key)           // 挪到末尾 = 标记为"最近使用"
```

这三行 `keys.delete + keys.add` 就是 **LRU 的全部实现**：`Set` 的迭代顺序是插入顺序，所以尾部是最近使用、头部是最久未用，超限就淘汰头部——**是严格 LRU，不是"先进先出"**。

而那两个 shapeFlag 是 KeepAlive 和渲染器之间的"暗号"：

| shapeFlag | 值 | 渲染器看到后做什么 |
|---|---|---|
| COMPONENT_SHOULD_KEEP_ALIVE | 256 | 卸载时**不真卸**，把 DOM 移进隐藏容器，触发 `deactivated` |
| COMPONENT_KEPT_ALIVE | 512 | 挂载时**不重新创建**，把 DOM 从隐藏容器移回容器，触发 `activated` |

#### 2.3 三个实测确认的坑

**坑一：`max` 淘汰时，不会触发 `deactivated`，直接 `unmounted`。**

实测 hook 顺序（切到第三个组件、`max: 2` 时，A 是最久未用的）：

```text
切到 C 时: A:unmounted → B:deactivated → C:mounted → C:activated
```

注意被淘汰的 A 是 **`unmounted` 而没有 `deactivated`**。为什么？因为淘汰发生在 KeepAlive 的渲染阶段（懒执行的 `setup` 里），此时 A 的 `SHOULD_KEEP_ALIVE` 标记会被 `resetShapeFlag` 摘掉再正常卸载。而 B 是曾经的活跃组件，它的 `deactivated` 是 patch 阶段卸载旧 vnode 时才触发的。

实践含义：**"离开时清理"的逻辑不能只写 `onDeactivated`**，被 `max` 淘汰或父组件整体销毁时走的是 `onUnmounted`。定时器、全局事件监听、WebSocket 这些要在两处都考虑。

**坑二：`include` / `exclude` 匹配的是组件名字，不是 `key`。**

```js
if ((include && (!name || !matches(include, name))) || (exclude && name && matches(exclude, name))) {
  vnode.shapeFlag &= ~256      // 摘掉 keep-alive 标记 → 这个组件切走就真销毁
  return rawVNode
}
```

这里的 `name` 来自 `getComponentName()`，它会**依次看 `Component.name` 和 `Component.__name`**（源码就是 `Component.name || Component.__name`）。而 `__name` 正是 **3.2.34+ 的 `<script setup>` 编译器根据文件名自动生成的那个名字**：

```js
// source: getComponentName（runtime-core）
function getComponentName(Component, includeInferred = true) {
  return Component.name || (includeInferred && Component.__name)
}
```

所以 `UserList.vue` 写 `<script setup>` 时天然能匹配 `include="UserList"`，**不用手写 `name`**。反过来，`include` 静默失效的原因通常是这几个：

- 组件是**手写的匿名组件**（`defineComponent({})` 不写 `name`、`h()` 直接创建），既没有 `name` 也没有 `__name`；
- 名字**大小写或拼写不一致**（`matches` 对字符串是**精确匹配**，不像 `:class` 那样宽松）；
- `<KeepAlive>` 包住的**不是组件**（比如一个纯 `<div>`），它压根不进入缓存分支。

排查方法：Vue DevTools 里看组件显示的名字，跟你写进 `include` 的字符串逐字对比。

**坑三：`<KeepAlive>` 只该有一个组件子节点。**

实测同时传两个子节点时会有 dev 警告：

```text
[Vue warn]: KeepAlive should contain exactly one component child.
```

它只对**组件**做缓存处理，非组件内容（纯元素）会原样渲染、不被拦截。所以 `<KeepAlive>` 里通常配 `<component :is>` 或 `<router-view>`。

#### 2.4 activated / deactivated 的触发时机

官方文档明确了两句话，面试可以直接引用：

- **`onActivated` 在组件挂载时也会调用**（首次渲染就触发一次）；
- **`onDeactivated` 在组件卸载时也会调用**。

还有一个容易漏的：**这两个钩子不只对缓存树里的根组件生效，对缓存树中的后代组件同样生效**。所以子组件的 `onActivated` 也会被触发，不需要一层层往外传事件。

#### 2.5 放在 `<KeepAlive>` 里的 `<Suspense>`

源码里有个特判，值得当冷知识记一下：

```js
return isSuspense(rawVNode.type) ? rawVNode : vnode
//     ↑ 如果子节点是 Suspense，返回原始 vnode（Suspense 自己管理内部子树）
```

因为 Suspense 的子树结构是它自己动态决定的，KeepAlive 硬套缓存的 vnode 会出问题。所以 `<KeepAlive><Suspense>...</Suspense></KeepAlive>` 能跑，但缓存行为由 Suspense 边界自己控制。

### 3. Suspense：至今仍是实验性 API

先记一条硬事实，很多八股文还在说"Vue 3 正式版已支持"——**Suspense 在 3.5 里仍然是实验性功能**，dev 环境会直接打印：

```text
<Suspense> is an experimental feature and its API will likely change.
```

面试被问到"生产能用吗"，正确答法是：**能用，但 API 可能变，官方未承诺稳定；要有兜底方案（比如自己用 loading 状态管理）。**

#### 3.1 它解决什么问题

```
组件树
  App
   └─ Suspense            ← 异步边界（SuspenseBoundary）
       ├─ default 插槽 → AsyncUserCard（async setup，等接口）
       └─ fallback 插槽 → <Skeleton />
```

Suspense 维护一个 **SuspenseBoundary**，核心状态是 **`deps`（未完成的异步依赖计数）**：

1. 挂载时先渲染 `fallback`；
2. 子树里的组件如果有 `async setup()`（含 `<script setup>` 顶层 `await`）或 `defineAsyncComponent`，它们的 Promise 会被 `registerDep` 注册到**最近的** Suspense 边界上，`deps++`；
3. 每个依赖 resolve 时 `deps--`；
4. `deps` 归零 → 一次性切到 `default` 分支渲染真正的子树；
5. 任一依赖 reject → **Suspense 不做兜底**：错误交给 `app.config.errorHandler`（或上层 `onErrorCaptured`），边界自己不会去显示另一个"错误插槽"，实测 DOM 最后只剩 `<!---->`。

第 5 条值得单独强调，因为它和很多人以为的"Suspense 会自动处理失败"相反。实测：`async setup` 里 `throw` 之后，`errorHandler` 收到了错误，页面最终是一个空的注释占位——**错误处理必须自己写**（比如把接口调用包在 `try/catch` 里，失败时正常 `return` 一个错误态组件）。

**关键点：是"一次性"切换。** 有多个异步依赖时，只要还有一个没完成，就继续显示 fallback，不会渲染"半成品"——这正是 Suspense 相比"每个组件各自 `v-if="loading"`"的价值：**避免页面出现多个错落的 loading 骨架闪烁**。

#### 3.2 三个容易答错的点

**① 没有 Suspense 包裹时，`async setup` 的组件根本不渲染。** 实测直接挂在页面上时，DOM 里只剩一个注释占位 `<!---->`，并且 dev 会警告：

```text
[Vue warn]: Component <Anonymous>: setup function returned a promise, but no <Suspense>
boundary was found in the parent component tree. A component with async setup() must be
nested in a <Suspense> in order to be rendered.
```

结论：**异步依赖必须有个边界去接，没边界就没人等它。** `async setup`（含 `<script setup>` 顶层 `await`）和 `<Suspense>` 是配套出现的，不能只写一个。这也是"为什么我写了顶层 await 页面一片空白"的官方答案。

**② 嵌套 Suspense 的行为由 `suspensible` prop 控制（Vue 3.3+）。**

```vue
<Suspense>
  <AsyncLayout>
    <!-- suspensible（默认 true）：内层边界把自己的异步依赖也上报给父边界
         → 父组件要等内外全部 resolve 才隐藏 fallback
         设为 false：内层自己管自己，父边界只等外层的依赖 -->
    <Suspense suspensible>...</Suspense>
  </AsyncLayout>
</Suspense>
```

**③ 和 Teleport 有个隐藏联动。** 实测源码里 Teleport 挂载时会检查 `parentSuspense && parentSuspense.pendingBranch`，成立就 `queuePendingMount` 延迟挂载。也就是说 **Teleport 在 Suspense 处于挂起状态时会等父边界解析完再挂**，避免往还没确定的目标里插东西。这个细节答出来基本可以收场了。

### 4. 三个组件的共同设计模式（面试收尾用）

| | Teleport | KeepAlive | Suspense |
|---|---|---|---|
| 内部标记 | `__isTeleport` | `__isKeepAlive` | `__isSuspense` |
| shapeFlag | TELEPORT (64) | 靠 vnode 上的 256 / 512 标记 | SUSPENSE (128) |
| 特殊能力 | 自定义挂载容器 | 拦截卸载 / 挂载 | 拦截子树渲染 |
| 是否产生 DOM | 只留两个注释锚点 | 不产生任何 DOM | 产生 fallback 或 default |
| SSR 特殊处理 | `__teleportBuffers` | 直接渲染子节点（不缓存） | hydration 时等已 resolve |
| 稳定状态 | 稳定 | 稳定 | **实验性** |

**统一答法**：这三个组件都是"框架内部能力的外露"——它们做的事超出了普通组件能做的范围（选容器、拦卸载、拦渲染），所以只能在渲染器里开特判口子。理解这一点，比背三个组件的 props 有用得多。

## 其实你每天都在用

- 用 Element Plus / Ant Design Vue 的 `<Dialog>`、`<Drawer>` 加 `append-to-body`，底层就是 Teleport 挂到 `body`。
- 后台管理系统的多标签页（`<router-view>` + `<KeepAlive>` + `include`），切回来输入框里的内容和滚动位置都还在，就是缓存 vnode 的效果。
- KeepAlive 里给菜单组件设 `max="10"`，打开超过 10 个页面后最久没回来的那个被淘汰——这个"静默卸载"就是坑一说的不触发 `deactivated`。
- 弹窗组件里 `onDeactivated` 里清定时器、`onUnmounted` 里也清一遍，就是被坑一教育过的写法。
- Nuxt / Vite SSR 项目里用 Teleport 时，服务端要处理 `__teleportBuffers`，不然 `<head>` 里少东西、弹窗位置错乱。
- 用 `defineAsyncComponent` 加载重型图表组件时外面套 `<Suspense>`，是为了让整页的 loading 只出现一次而不是图表区域闪三次。
- Vue DevTools 里看到的"组件树位置"和"DOM 树位置"不一致的场景，基本都是 Teleport。

## 常见误解（FAQ）

**❌ 误区1："Teleport 会让组件脱离父组件，父组件的 provide 就 inject 不到了"**

错。Teleport 只改变 **DOM 挂载位置**，组件树的父子关系完全不变。`provide` / `inject` 是按组件树（`parent` 链）查找的，所以**依然能注入**；`emit`、props、响应式、DevTools 的组件层级也都正常。真正会"断链"的是那些按 DOM 树工作的东西（比如 CSS 继承、事件按真实 DOM 冒泡的顺序）。

**❌ 误区2："KeepAlive 缓存的 key 是组件的 name"**

错。源码是 `vnode.key == null ? comp（组件定义对象） : vnode.key`。**`name` 只用于 `include` / `exclude` 匹配。** 顺带纠正另一个版本："缓存的是组件实例"也不精确——**缓存的是 vnode**，组件实例挂在 `vnode.component` 上被顺带留住。

**❌ 误区3："KeepAlive 里的组件切走时不会执行任何卸载相关钩子"**

`onDeactivated` 会执行（这是你清理副作用的正确位置），但 `onUnmounted` / `onBeforeUnmount` **不会**——这是它设计上要的效果。反向的坑更值得记：**当组件被 `max` 淘汰或被父组件一起销毁时，走的是 `unmounted` 而不会补一次 `deactivated`**（实测确认）。所以清理逻辑两处都要写，或者统一抽成一个函数。

**❌ 误区4："KeepAlive 的 max 是先进先出（FIFO）"**

是 **LRU**。命中缓存时会执行 `keys.delete(key); keys.add(key)` 把 key 挪到末尾，超限时淘汰的是 `keys` 的**第一个**（最久未用）。如果实现成 FIFO，一个高频访问的老页面会莫名其妙被淘汰掉。

**❌ 误区5："Suspense 已经稳定了，可以放心用于生产"**

3.5 里它**仍是实验性功能**，dev 会打印 `<Suspense> is an experimental feature and its API will likely change.`。而且配套限制不少：`async setup` 必须被 Suspense 包住才渲染；`suspensible` 是 3.3 才有的；SSR + Suspense + 异步组件的 hydration 时机有专门的修复历史。回答时说"实验性"比说"稳定"安全得多。

**❌ 误区6："Suspense 的 fallback 是每个异步子组件各自 loading"**

不是。Suspense 的语义是**边界级别的等待**：边界内**所有**异步依赖都 resolve 之后才一次性渲染 `default` 子树，期间一直显示 `fallback`。这正是它相对"每个区块各自 `v-if="loading"`"的优势——不会出现多个骨架屏错落闪动。

**❌ 误区7："Teleport 的 `to` 只能用 CSS 选择器字符串"**

`to` 也可以是**一个真实的 DOM 元素**（`ref` 拿到的 `el`，或 `document.body`）。实测两种传法都能正常传送。另外 `defer`（3.5+）能让目标由同一轮渲染中靠后的组件提供，解决了"目标还不存在"的经典报错；`disabled` 则用于响应式地在"传送"和"原地渲染"之间切换（移动端弹窗改内联场景）。

## 一句话总结

**Teleport 管「渲染到哪」（只挪 DOM 不动组件树），KeepAlive 管「卸载时说声别卸」（缓存 vnode，LRU 靠 Map + Set），Suspense 管「什么时候渲染」（边界级等待异步依赖）——三者的共同点是：只有渲染器才有权限做的事，才会被做成内置组件。**
