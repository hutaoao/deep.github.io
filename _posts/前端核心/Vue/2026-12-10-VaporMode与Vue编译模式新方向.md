---
layout: post
title: "Vapor Mode 与 Vue 编译模式新方向：把虚拟 DOM 整个删掉"
date: 2026-12-10 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [Vapor Mode, 编译模式, 虚拟DOM, 细粒度更新, renderEffect]
description: >
  Vapor Mode 是 Vue 3.6 引入的全新编译模式：编译期直接把模板变成"一次建 DOM + 精确改 DOM"的代码，
  运行时没有 VNode、也没有 diff。文章用实测编译产物对比同一模板的 VDOM 版与 Vapor 版，
  讲清它省掉了什么、拿什么替代、代价是什么；并含 rc.2 那个真实 BREAKING 变更
  （事件委托从默认改为显式 .delegate）、不支持特性清单与面试口述稿。
---

## 一句话概括

**Vapor Mode 是 Vue 3.6 引入的一种新编译模式：编译期就把模板直接编译成"建 DOM + 改 DOM"的代码，运行时不再创建 VNode，也不再做 diff。**

它回答的是那个被问了十年的问题——"虚拟 DOM 是不是纯开销"。Svelte 5、Solid 这些编译型框架早就选了"不要 VDOM"，现在 Vue 也给出了这条路，而且**不换语法、不重写组件**。

面试为什么问它？因为它不是某个 API 的增删，而是 Vue 渲染架构的第一次真正变动。能把这道题答好，说明你不是只会用 `ref` 和 `computed`：

- 你得能说清 **VDOM 到底贵在哪**（不是"diff 慢"，是每层都要建对象）；
- 你得能说清 **Vapor 拿什么替代 diff**（编译期已知的"状态 → 节点"映射）；
- 你还得说出 **代价**（只支持一部分 API、组件边界上的坑、生态兼容）。

先说版本状态，这点很容易被博客写错：**Vapor Mode 在 Vue 3.6 里，目前仍是 RC**（写作时最新是 `3.6.0-rc.9`，2026-09-18），官方口径是 feature-complete，推荐的使用场景只有两个——**在存量项目里给某个性能敏感的页面用**，或者**用纯 Vapor 写小型新应用**。所以面试时正确的姿势是"我知道它怎么工作、我知道什么时候该上"，而不是"我们项目已经全量用了"。

## 核心知识点

### 1. 先算账：Vapor 砍掉的是中间两步，不是"把 diff 优化得更快"

传统 Vue 组件的完整链路是五步：

```
解析模板 → 编译成 render 函数 → 执行 render 得到 VNode 树 → diff 新旧两棵树 → patch 真实 DOM
```

Vapor 是三步：

```
解析模板 → 编译成直接操作 DOM 的代码 → 状态变了，直接改对应的 DOM 节点
```

很多人第一反应是"diff 能有多慢"。其实 VDOM 的成本大头不在"比较算法"，而在**它必须先造出一棵完整的对象树**：每渲染一次，每个元素、每个文本、每个 prop 都对应一个 JS 对象，这些对象要分配内存、要被 GC 回收，然后才有资格进入 diff。节点数一多，光"造树 + 扔树"就是一笔固定开销。

Vapor 的思路不是把 diff 写得更聪明，而是：**编译期就已经知道"哪个状态对应哪个 DOM 节点"，运行时为什么还要再算一遍？**

一句话记住：**Vapor 不是更快的 diff，是没有 diff。**

### 2. 同一段模板，两种产物（实测对比，这是最能说明问题的一张图）

用 `@vue/compiler-sfc` 对下面这段模板编译两次——一次普通编译，一次加 `vapor` 标记：

{% raw %}
```vue
<template>
  <div class="card">
    <h2>{{ title }}</h2>
    <p>静态文本，永远不变</p>
    <button @click="inc">点了 {{ count }} 次</button>
    <span :class="{ active: count > 3 }">状态</span>
    <input v-model="title" />
  </div>
</template>
```
{% endraw %}

**VDOM 版产物**（节选）：

```text
return (_openBlock(), _createElementBlock("div", { class: "card" }, [
  _createElementVNode("h2", null, _toDisplayString(_ctx.title), 1 /* TEXT */),
  _createElementVNode("p", null, "静态文本，永远不变"),
  _createElementVNode("button", { onClick: _ctx.inc },
    "点了 " + _toDisplayString(_ctx.count) + " 次", 9 /* TEXT, PROPS */, ["onClick"]),
  /* ... */
]))
```

**Vapor 版产物**（完整）：

```js
import {
  child as _child, nthChild as _nthChild, next as _next, txt as _txt,
  on as _on, toDisplayString as _toDisplayString, setText as _setText,
  setClassName as _setClassName, renderEffect as _renderEffect,
  applyTextModel as _applyTextModel, template as _template,
} from 'vue'

const t0 = _template(
  '<div class=card><h2> </h2><p>静态文本，永远不变</p><button> </button><span>状态</span><input>', 1
)

export function render(_ctx, $props, $emit, $attrs, $slots) {
  const n4 = t0()
  const n0 = _child(n4)        // 第 1 个子节点：h2
  const n1 = _nthChild(n4, 2)  // 第 2 个子节点：button
  const n2 = _next(n1)         // button 的下一个兄弟：span
  const n3 = _next(n2)         // 再下一个：input
  const x0 = _txt(n0)
  const x1 = _txt(n1)
  _on(n1, 'click', e => _ctx.inc(e))
  _renderEffect(() => {
    const _count = _ctx.count
    _setText(x0, _toDisplayString(_ctx.title))
    _setText(x1, '点了 ' + _toDisplayString(_count) + ' 次')
    _setClassName(n2, (_count > 3 ? 1 : 0), 'active')
  })
  _applyTextModel(n3, () => _ctx.title, _value => (_ctx.title = _value))
  return n4
}
```

从这两份产物能读出四个关键结论：

1. **Vapor 产物里完全没有 `createElementBlock` / `createElementVNode`**——一个 VNode 都不建。
2. **整个静态骨架被压成了一个字符串**交给 `_template()`。看运行时源码，`template()` 第一次调用时把这段 HTML 塞进一个缓存的 `<template>` 元素解析一次（其实就是我们手写组件时用的同一个技巧），之后**每个组件实例只做一次 `cloneNode(true)`**。VDOM 版是每个元素一次 `createElement` 逐层拼装，Vapor 版是"一次性克隆 + 游标定位"。
3. **动态位置在字符串里是空的占位**（`<h2> </h2>` 里那个空格），运行时靠 `_child()` / `_nthChild()` / `_next()` 走指针找到它们——顺序在编译期就固定了，不需要运行时再查。
4. **纯静态的 `<p>` 在产物里连引用都没有**。静态节点被"埋"在字符串里，从此再也不会被碰。

### 3. 更新路径：render 只跑一次，这是 Vapor 最反直觉的地方

上面那段产物里，动态绑定全被包在一个 `_renderEffect()` 里。它意味着：**组件函数只在挂载时执行一次，之后状态变化完全不经过 render。**

实测（`vue@3.6.0-rc.9` + jsdom，用 `createVaporApp` 挂载）：

```
初始 HTML: <div><p>A</p><span>B</span></div>
render 函数调用次数: 1
改 a 之后 HTML: <div><p>A2</p><span>B</span></div>
render 函数调用次数: 1        ← 没有重新渲染
同一节点被就地更新: true       ← 文本节点身份没变，只改了 nodeValue
```

对照一下 VDOM 的思路会更清楚：

| | VDOM | Vapor |
|---|---|---|
| 状态变化时 | 重跑 render，产出新 VNode 树 | 不重跑 render |
| 怎么找到要改的节点 | diff 新旧树后得出 | 编译期就已经知道，直接引用 |
| 改的方式 | patch 到真实 DOM | `setText` / `setProp` / `setClassName` 直写 |

**这里要纠正一个流传很广的说法。** 很多文章写"Vapor 给每个响应式绑定生成一个独立的 effect"。实测并非如此——**同一个渲染作用域内的所有动态绑定会被合并进一个 `renderEffect`**。我上面那段产物里，`title`、`count`、`:class` 三个绑定全在同一个 effect 里。真正的拆分发生在"作用域边界"上：

- `v-for` 的**每一项**是一个独立作用域，项内绑定有自己独立的 `renderEffect`（这样改一项不会牵动整个列表）；
- **插槽内容**是独立作用域；
- `v-once` 的内容**根本没有 effect**，赋值只在创建时跑一次。

可以这么记：**Vapor 的更新粒度是"渲染作用域"，不是"每一个绑定"。**

### 4. props 变成了 getter——这是细粒度更新能成立的前提

看 Vapor 编译产物里怎么传 props：

```js
_createAssetComponent('Child', {
  msg: () => (_ctx.title),        // 注意不是 msg: _ctx.title
  onChange: () => _ctx.onChange,
}, () => { /* 插槽内容，独立作用域 */ })
```

**传下去的不是值，是一个"要的时候自己去取"的函数。**

这一个小改动解决的是 VDOM 里最贵的那部分工作：以前父组件重渲染 → 子组件拿到新 props 对象 → 比较 props 决定要不要更新。现在子组件在读取 `props.msg` 时直接读到父组件作用域里的响应式数据，**依赖关系直接连到子组件自己的 renderEffect 上**——父组件根本不需要"把 props 推下去"。

面试时这句是加分项：**Vapor 里组件更新边界的含义变了——不是"父组件重渲染带动子树"，而是"谁依赖了谁，就只更新谁"。**

### 5. v-if / v-for 变成了结构指令，而不是 diff 的分支

实测产物：

```js
// v-if / v-else
const n0 = _createIf(() => (_ctx.ok),
  () => t0(),   // true 分支的工厂
  () => t1(),   // false 分支的工厂
  357 /* TRUE_SINGLE_ROOT, FALSE_SINGLE_ROOT, TRUE_NO_SCOPE, FALSE_NO_SCOPE, KEYED_INDEX_0 */
)

// v-for
_createFor(() => (_ctx.list), (_for_item0) => {
  const n2 = t0()
  const x2 = _txt(n2)
  _renderEffect(() => _setText(x2, _toDisplayString(_for_item0.value.name)))
  return n2
}, (i) => (i.id), 9 /* FAST_REMOVE, IS_SINGLE_NODE */)

_setInsertionState(n3)   // 有结构指令时才生成，用来记插入位置（锚点）
```

VDOM 里"该删哪个、该插哪里"是 diff 对比出来的结论；Vapor 里它是**直接建 / 直接删**，那些 `flags` 位是编译期算好的静态信息（是不是单根、有没有作用域、key 用什么），运行时不用再猜。

注意最后那个 `_setInsertionState`：**只有模板里出现结构指令时才会生成**。它是给条件/列表分支用的"锚点"，用来保证切换分支时新节点插在对的位置——这类细节恰好说明 Vapor 不是"删掉了 diff 那么简单"，而是把原来 diff 承担的一部分职责平摊到了编译期和运行时指令上。

### 6. 怎么开启：三种标记 + 两种应用入口（以及一个网上抄错的配置）

**组件级开启（三选一，编译期等价）：**

```vue
<script setup vapor>
// ...
</script>
```

{% raw %}
```vue
<script vapor>
// ... 是 <script setup vapor> 的简写
</script>

<template vapor>
  <!-- ... 加在 template 上，整块 SFC 走 Vapor 编译 -->
</template>
```
{% endraw %}

**应用入口两种：**

```js
// ① 纯 Vapor 应用：不会把 VDOM 运行时打进包里，基础体积最小
import { createVaporApp } from 'vue'
createVaporApp(App).mount('#app')

// ② 混用：在普通 createApp 实例里使用 Vapor 组件，必须装 interop 插件
import { createApp, vaporInteropPlugin } from 'vue'
createApp(App).use(vaporInteropPlugin).mount('#app')
```

关于 ② 有两个官方提醒值得记：装 `vaporInteropPlugin` 会把 VDOM 运行时拉回来，**一部分体积收益就抵消掉了**；而且混合嵌套目前"覆盖 props / 事件 / 插槽的标准用法，但没覆盖所有边界情况"。所以官方的建议是——**把两种模式分成各自成片的区域，别逐组件交错**。

**这里有个坑，很多教程抄错了：** 网上流传的开启写法是 `vue({ template: { compilerOptions: { vapor: true } } })`。实测这个配置**在 `@vue/compiler-dom` 上完全不生效**——我给它传了 `vapor: true`，产出的依旧是 `createElementBlock` 那套 VDOM 代码（翻源码也确认了，`compiler-dom` 里根本没有 `vapor` 这个选项）。真正的路由发生在 `@vue/compiler-sfc`：它读 SFC 上的 `vapor` 标记（`descriptor.vapor = hasAttr(node, "vapor")`），然后**把编译器整个换成 `@vue/compiler-vapor`**。所以记忆点很简单：**Vapor 是 SFC 编译器的选择，不是某个插件的开关。**

### 7. 明确不支持的清单（别猜，这是官方原表）

| 不支持 / 不适用 | 原因（一句话） |
|---|---|
| Options API | Vapor 的编译产物围绕 `setup` 作用域，没有 `this` 那一套 |
| `app.config.globalProperties` | 同上，模板里的标识符按作用域解析，不走实例代理 |
| `getCurrentInstance()` | 在 Vapor 组件里**返回 `null`**（我实测确认过） |
| `@vue:xxx` 元素级生命周期事件 | 依赖 VNode 的生命周期钩子 |
| `v-memo` | 它的作用是"跳过 diff"，而 Vapor 没有 diff 可跳 |
| 组件模板 ref 上的 `$el`/`$props`/`$attrs`/`$slots`/`$refs` | 这些都是"组件公共实例代理"上的东西 |
| 自定义指令 | 接口换了（见下） |

自定义指令在 Vapor 里不是原来的对象钩子形态，而是一个函数：

```ts
type VaporDirective = (
  node: Element | VaporComponentInstance,
  value?: () => any,       // 注意是 getter，要调用才拿到值
  argument?: string,
  modifiers?: DirectiveModifiers,
) => (() => void) | void   // 可以返回一个清理函数
```

一句话概括这整张表：**凡是依赖"VNode"或"组件公共实例"的特性，Vapor 都没有。** 这不是没做完，是设计上就不打算做——因为这两样东西正是它要删掉的。

### 8. 事件委托：从"默认委托"改成了"显式 `.delegate`"（一个真实的 BREAKING 变更）

这是我用这题踩到的活教材，特别值得在面试里讲。

- **3.6.0-rc.1 的发布说明**（2026-07-18）写的是：Vapor 会把合格的事件**委托到 `document`**，并警告"如果祖先调用了 `stopPropagation()`，事件到不了 document，委托的处理器就不会执行"。
- **3.6.0-rc.2 的 BREAKING CHANGES**（2026-07-22）直接把这条改了：**Vapor 事件委托改为 opt-in**。原文理由是"为了让 Vapor 与标准 Vue、与原生 DOM 的事件行为一致"——因为委托会造成"祖先 `stopPropagation()` 让子组件处理器不执行"这种和 VDOM 不一样的行为。同时 `compilerOptions.eventDelegation` 这个选项**被移除了**。

所以要委托得显式写 `.delegate`：

{% raw %}
```vue
<button @click.delegate="onClick" />
```
{% endraw %}

实测两份产物对比，差别非常直观：

```js
// @click  → 默认：直接绑在元素上
_on(n0, "click", e => _ctx.fn(e))

// @click.delegate → 走 document 委托
_delegateEvents("click")
n0.$evtclick = _createInvoker(e => _ctx.fn(e))
```

配套细节（都是从 `@vue/compiler-vapor` 源码里读出来的）：

- **委托只对白名单事件生效**：`beforeinput / click / dblclick / contextmenu / focusin / focusout / input / keydown / keyup / mousedown / mousemove / mouseout / mouseover / mouseup / pointerdown / pointermove / pointerout / pointerover / pointerup / touchend / touchmove / touchstart`。写 `@scroll.delegate` 会警告 `.delegate modifier is not supported on the "scroll" event. The listener will be attached directly.` 然后回退成直接绑定。
- **`.delegate` 不能和事件选项修饰符叠加**。`@click.delegate.stop` 实测产物是 `_on(n0, "click", _withModifiers(...))`——又回到直接绑定了。源码里的判定是 `delegate = isDelegatableEvent && !eventOptionModifiers.length && !hasStopHandler`。
- 编译器只在**有组件用了委托**时才生成 `_delegateEvents("click")` 这一行，全项目同一个事件只装一个 document 监听。

**这题真正的面试价值不在 API，而在于它示范了一件事：涉及版本特性时必须以 changelog 为准。** 半年前的文章说"Vapor 默认委托到 document"，现在已经是错的了。

### 9. Vapor 和现有的编译优化（静态提升 / patchFlag / Block Tree）是什么关系

这是最常见的追问，答不好会显得"只会背概念"。用一句话把两条路线分开：

- **静态提升、patchFlag、Block Tree 都是"在保留 diff 的前提下，把 diff 变便宜"**：静态提升省掉重复创建；patchFlag 把"这个节点有什么会变"编译进 VNode，省掉逐属性对比；Block Tree 把 diff 的范围从整棵树收窄到动态节点组成的扁平数组。
- **Vapor 是把整条路换掉**：不需要标记动态部分，因为根本不遍历；不需要 Block Tree，因为本来就不建树。

所以它不是"更高级的 patchFlag"，而是**另一条技术路线**。这解释了几个现象：

- 为什么 `v-memo` 在 Vapor 里不支持——它本来就是给 diff 做减法的工具，没有 diff 就没有意义；
- 为什么 Vapor 组件和 VDOM 组件的优化手段**互不适用**（在 Vapor 组件上写 `v-memo` 不会更快，只会报错）；
- 为什么官方强调"分区域"——两种模式的边界上，优化假设是不通用的。

**一句可以直接背的话术：** "patchFlag 是在给 diff 做减法，Vapor 是把 diff 删掉。前者能兼容存量代码，后者要求组件按新规则写。"

### 10. 面试怎么答

**30 秒版（被问到"了解 Vapor 吗"时）：**

> Vapor 是 Vue 3.6 的编译模式，编译期直接把模板编译成操作 DOM 的代码，运行时既不建 VNode 也不 diff。它是组件级 opt-in 的，同一套 Composition API 不用改；代价是只支持一部分 API，Options API、`getCurrentInstance`、`v-memo` 这些依赖 VNode 或组件实例的特性都没有。目前还是 RC，官方建议用在性能敏感页面或小型新项目。

**3 分钟版（被追问原理时，按这个顺序讲）：**

1. **省的是什么**——VDOM 每渲染一次都要造一棵对象树再扔掉，diff 只是后面的步骤；Vapor 让编译期算好"状态 → 节点"的映射，运行时直接改 DOM。
2. **编译产物长什么样**——整个静态骨架压成一个字符串，用缓存的 `<template>` 解析一次、每个实例 `cloneNode` 一次；动态位置靠 `child/nthChild/next` 定位；动态绑定包在一个 `renderEffect` 里。
3. **更新怎么发生**——组件函数只在挂载时跑一次，之后状态变化只触发 renderEffect，直写文本节点 / 属性 / class。
4. **组件间怎么协作**——props 以 getter 形式传下去，子组件读 prop 就等于在自己的 effect 里订阅了父组件的数据，所以没有"父组件重渲染带动子树"这回事。
5. **代价**——API 子集、生态组件库都是 VDOM 组件需要 interop、混合嵌套有边界情况。

## 其实你每天都在用

- 你在模板里写 `<div v-once>{ { x } }</div>`，本质就是在**手动做 Vapor 编译器自动做的事**——把这棵子树彻底排除在更新之外。
- 你的长列表卡顿，第一反应是上虚拟滚动；Vapor 是另一条路，它**压低每行更新的成本，但不管 DOM 数量**——节点太多照样得靠虚拟滚动解决。
- 你用 `patchFlag` / Block Tree 的直觉判断"这个组件会不会重渲染"，在 Vapor 组件上基本失效；反过来，`v-memo` 这种"手动挡"优化在 Vapor 里直接不可用。
- 你封装的工具函数里如果调了 `getCurrentInstance()`（比如自己写指令、写库），**在 Vapor 组件里会拿到 `null`**——这就是"subset"落到项目里的真实代价。
- 你项目里的 Element Plus / Ant Design Vue 全都是 VDOM 组件，所以混合边界上的问题**会先出现在它们身上**，这也是官方说"按区域划分、别交错"的原因。
- 你在 Vite 配置里给 `vue()` 插件传的每个选项，今后都有可能和"编译模式"挂钩——但别抄那些 `compilerOptions: { vapor: true }` 的教程，实测无效。

## 常见误解（FAQ）

**❌ 误区1："Vapor 就是 Vue 4，得重写组件。"**

不是。它是**组件级 opt-in** 的编译模式，同一份 Composition API、同一套模板语法，只多一个 `vapor` 标记。而且官方明确建议先在存量项目里挑一个性能敏感页面用，而不是全盘迁移。

**❌ 误区2："Vapor 把 diff 换成了更快的 diff 算法。"**

没有 diff 了。它不比较新旧树，因为根本没有树。这也是为什么 `v-memo`（一个专门用来"跳过 diff"的工具）在 Vapor 里没有存在意义。

**❌ 误区3："加了 `vapor` 就一定更快。"**

收益主要来自**更新密集 + 组件数量多**的场景。首次渲染的提升相对有限（DOM 创建本身还是要做），而在"静态内容为主、几乎不更新"的页面上，普通 Vue 的静态提升已经够用了。官方口径也克制：`createVaporApp` 的最大价值是**不把 VDOM 运行时打进包里的那部分体积**。所以正确说法是"降低基准开销"，不是"无差别加速"。

**❌ 误区4："Vapor 给每个响应式绑定各生成一个 effect。"**

实测不是。**同一渲染作用域里的所有动态绑定合并进一个 `renderEffect`**；只有跨越作用域边界时才会拆开——`v-for` 的每一项、插槽内容各自拥有独立作用域，`v-once` 则完全不生成 effect。

**❌ 误区5："Vapor 组件里照样能用 `getCurrentInstance()`、Options API、`v-memo`。"**

全都不行。`getCurrentInstance()` 在 Vapor 组件里返回 **`null`**（实测确认）；Options API 不支持；`v-memo` 不支持；`@vue:xxx` 元素级生命周期事件不支持；组件模板 ref 上也读不到 `$el` / `$props` / `$attrs` / `$slots` / `$refs`。**一条判断规律：依赖 VNode 或组件公共实例的特性，Vapor 一律没有。**

**❌ 误区6："Vapor 会把事件自动委托到 document。"**

这是 **3.6.0-rc.1 的旧行为**。从 **3.6.0-rc.2 起，委托改成了显式 opt-in**：默认直接绑在元素上，要委托得写 `@click.delegate`，而且 `compilerOptions.eventDelegation` 这个选项已经被移除。官方给的理由就是行为一致性——委托会导致"祖先 `stopPropagation()` 让子组件处理器不执行"。

**❌ 误区7："在 vite 里配 `vue({ template: { compilerOptions: { vapor: true } } })` 就能全过程开启。"**

实测无效——`@vue/compiler-dom` 根本没有 `vapor` 这个选项，传了也照样产出 `createElementBlock`。真正的开关是 SFC 上的 `vapor` 标记，由 `@vue/compiler-sfc` 负责把编译器切换成 `@vue/compiler-vapor`。

## 一句话总结

**Vapor 不是把虚拟 DOM 优化得更快，而是让编译器提前把"状态变了该改哪个节点"算好，运行时直接改——省下的不是算法，是整个中间层。**
