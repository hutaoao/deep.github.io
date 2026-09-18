---
layout: post
title: "v-model 与 Transition 动画原理：两条模板语法糖的落地路径"
date: 2026-12-09 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [v-model, Transition, 模板编译, 动画原理, FLIP]
description: >
  把 v-model 和 <Transition> 放在一起讲，因为它们回答的是同一个问题——Vue 怎么把模板里的语法糖
  落到运行时：v-model 编译成"一个 prop + 一个事件"的显式协议，<Transition> 编译成一个普通组件、
  靠 vnode.transition 钩子被 patch 流程回调。含各表单元素编译产物对比、修饰符实测行为、
  六个过渡类名的完整时序、动画结束的判定机制与 TransitionGroup 的 FLIP 实现。
---

## 一句话概括

`v-model` 和 `<Transition>` 表面上是两个不相关的知识点，但它们回答的是同一个问题：**Vue 怎么把模板里的"语法糖"落到运行时？**

答案恰好是两条不同的路：

- **`v-model` 是纯语法糖**——编译期就被拆掉，变成一个 prop + 一个事件监听。运行时根本不知道"v-model"这三个字存在过。
- **`<Transition>` 不是语法糖**——它就是一个普普通通的**函数式组件**（内部包了一层 `BaseTransition`）。元素进出场之所以能加类名，是因为渲染器在 patch 时发现 `vnode.transition` 上有钩子，主动调了一次。

面试为什么问这两个？因为它们各自能带出一串高频追问：`v-model` 那条线问"编译产物是什么、为什么组件上要用 `modelValue`、多个 `v-model` 怎么实现"；`<Transition>` 那条线问"类名加在什么时机、Vue 怎么知道动画结束、`v-leave-active` 为什么要配 `position: absolute`"。

## 核心知识点

### 1. v-model 的本质：一个 prop + 一个事件（但编译产物跟传说中不一样）

流传最广的说法是"`v-model` 就是 `:value` 加 `@input`"。这话在 Vue 2 对，**在 Vue 3 已经不准了**。用 `compile` 实跑一下 Vue 3.5 的产物：

```js
// <input v-model="msg"> 的编译结果
_withDirectives((_openBlock(), _createElementBlock("input", {
  "onUpdate:modelValue": $event => ((_ctx.msg) = $event)   // 只有事件，没有 :value
}, null, 8 /* PROPS */, ["onUpdate:modelValue"])), [
  [_vModelText, _ctx.msg]                                   // 靠指令把值写进 DOM
])
```

两个关键差异：

1. **`value` 不在 props 里**，而是由 `vModelText` 指令的 `mounted` 钩子直接 `el.value = value == null ? '' : value` 写进去的（实测传 `null` 时 DOM 上是空字符串）。
2. 事件名统一成 **`onUpdate:modelValue`**，由指令内部挂的原生 `input` 监听器去触发它。

所以更准确的说法是：**Vue 3 里 `v-model` 是"一个 `onUpdate:modelValue` 回调 + 一个负责读写 DOM 的指令"**，指令负责处理各表单元素的差异，回调负责写回数据。

### 2. 不同表单元素，编译出来的指令不一样

这是面试很爱考的细节——`v-model` 会根据元素类型自动选不同的指令：

| 模板 | 编译出的指令 | 指令在做什么 |
| --- | --- | --- |
| `<input v-model="x">` / `<textarea>` | `vModelText` | 监听 `input`，直接写 `el.value` |
| `<input type="checkbox" v-model="x">` | `vModelCheckbox` | 监听 `change`，维护数组的 push/splice |
| `<input type="radio" v-model="x">` | `vModelRadio` | 监听 `change`，用 `value` 比对决定 `checked` |
| `<select v-model="x">` | `vModelSelect` | 监听 `change`，从 `el.options` 里筛出被选中的项 |

所以"所有 `v-model` 都编译成 `:value + @input`"这个说法至少在三种元素上是错的：checkbox / radio 绑的是 `checked` 而不是 `value`，select 读的是 `options` 里选中的项而不是 `value`。

绑定类型也有讲究，一个坑是 checkbox：

```js
// ❌ 想收集多选，却给了字符串——实测勾选后 model 直接变成 true
const hobbiesBad = ref('')

// ✅ 多选绑数组，Vue 自己 push/splice
const hobbies = ref([])
```

### 3. 修饰符：三个内置修饰符到底做了什么

修饰符不是"给个布尔值让组件自己看着办"——它们被编译进了指令的第四个参数：

```js
// <input v-model.trim.number.lazy="msg"> 的产物片段
[_vModelText, _ctx.msg, void 0, { trim: true, number: true, lazy: true }]
```

实测它们对输入的加工结果（这部分自己跑一遍最靠谱）：

| 修饰符 | 输入 | 拿到的值 |
| --- | --- | --- |
| 无 | `'  12  '` | `'  12  '`（原样） |
| `.trim` | `'  12  '` | `'12'` |
| `.number` | `'12'` | `12`（number） |
| `.number` | `''` | `''`（**空字符串，不是 0**） |
| `.number` | `'abc'` | `'abc'`（转不动就原样返回） |
| `.number` | `'12abc'` | `12`（**不是 `NaN`**，因为走的是 `parseFloat`） |

`.number` 那两个反直觉结果值得单独记住：**空串保持空串、`'12abc'` 会变成 `12`**。原因是它走的是 `looseToNumber`（`parseFloat` 语义）而不是 `Number()`：

```js
// runtime-dom：修饰符的实际处理（照抄源码）
function castValue(value, trim, number) {
  if (trim) value = value.trim()
  if (number) value = looseToNumber(value)
  return value
}

// 关键就在这一行：转不动就原样返回
const looseToNumber = (val) => {
  const n = parseFloat(val)
  return isNaN(n) ? val : n      // 'abc' → 'abc'；'12abc' → 12
}
```

`.lazy` 则是换事件：把 `input` 换成 `change`，输入过程中不同步，失焦或回车才同步——大表单里最实用的一个修饰符。

还有一个源码里的隐藏福利：**`<input type="number">` 会自动按数字处理**，不用再写 `.number`。

```js
// vModelText 内部
const castToNumber = number || (vnode.props && vnode.props.type === 'number')
```

顺带一个每天都会碰到的机制：`v-model` 的内部监听器会检查 `e.target.composing`，**中文输入法拼音还没上屏时不会同步**，等 `compositionend` 才写回数据。所以你不用自己处理输入法，Vue 已经拦在前面了。

### 4. 组件上的 v-model：从"隐式约定"变成"显式协议"

组件上的 `v-model` 编译产物很好认，就是一个 prop 加一个事件：

```js
// <MyComp v-model="msg" /> 的产物
_createBlock(_component_MyComp, {
  modelValue: _ctx.msg,                                     // ↓ 去
  "onUpdate:modelValue": $event => ((_ctx.msg) = $event)    // ↑ 回
}, null, 8, ["modelValue", "onUpdate:modelValue"])
```

子组件只要声明这两个东西就行：

```js
// 子组件：最朴素的写法
defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
```

实际项目里更常用 computed 的 getter/setter 中转（这样不用把 `props.modelValue` 到处传）：

```js
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])

const value = computed({
  get: () => props.modelValue,
  set: (v) => emit('update:modelValue', v)   // 子组件不改 prop，只发事件
})
```

**这里有个高频追问：`emits` 里到底要不要写 `update:modelValue`？**

不写也能跑，而且**不会有任何警告**——网上流传的"不声明会报 Missing required emit"是错的。原因在源码里：警告的条件是"事件既不在 `emits` 里，也没有对应的 `onXxx` prop"，而 `v-model` 恰好把 `onUpdate:modelValue` 当作 prop 传下来了，条件不成立。

但**还是要写**，因为实测差别在 `$attrs` 上：

```text
emits 未声明 → $attrs 键: ["onUpdate:modelValue"]   ← 会 fallthrough 到根元素
emits 已声明 → $attrs 键: []                        ← 被消费掉了
```

不声明的话，这个监听器会顺带挂到子组件根元素上，遇到"根元素恰好也派发同名事件"或"根是 Fragment"的情况就会出乱子。**声明 `emits` 的真正作用是把这个监听器从 `$attrs` 里拿走。**

多个 `v-model` 就更直白了，就是多组 prop + 事件：

```text
<MyComp v-model:title="t" v-model:content="c" /> 的产物：
{ title: _ctx.t,   "onUpdate:title":   $event => ((_ctx.t) = $event),
  content: _ctx.c, "onUpdate:content": $event => ((_ctx.c) = $event) }
```

修饰符也会沿着命名约定透传，prop 名是 **`modelModifiers`**（默认可写）或 **`<名>Modifiers`**（具名）：

```text
<MyComp v-model:title.cap="t" /> 的产物：
{ title: _ctx.t, "onUpdate:title": $event => ((_ctx.t) = $event),
  titleModifiers: { cap: true } }
```

子组件拿到 `titleModifiers.cap` 就知道要"首字母大写"——组件库支持自定义修饰符（格式化、去空格、转大小写）靠的就是这个约定。Vue 3.4+ 的 `defineModel()` 只是把这套协议收成一个可写 ref，编译产物完全一致。

### 5. `<Transition>` 的骨架：一个函数式组件，钩子挂在 vnode 上

`<Transition>` 的真身只有一行：

```js
// runtime-dom
const Transition = (props, { slots }) => h(BaseTransition, resolveTransitionProps(props), slots)
```

也就是说它是个**无状态的函数式包装**，真正干活的是 runtime-core 的 `BaseTransition`。它做的事可以概括成"往 children 的 vnode 上挂 `transition` 钩子"：

```text
<Transition>
  └─ BaseTransition：把 css 类名钩子 + 用户 JS 钩子合成一份
       └─ 写到子 vnode.transition = { beforeEnter, enter, leave, ... }
            ↓
       渲染器 patch 时发现 vnode.transition.beforeEnter，就调一下
```

所以它**不是语法糖**——编译器和运行时都不认识 `<Transition>`，它跟 `<KeepAlive>`、`<Teleport>` 一样是"内置组件"，区别是后两者需要渲染器特判，而 `<Transition>` 完全靠标准的 `vnode.transition` 字段工作。

一个直接结论：**它只接受单个子元素/单根组件**，多了一个就警告：

```text
[Vue warn]: <transition> can only be used on a single element or component.
Use <transition-group> for lists.
```

### 6. 六个类名的时序（实测时间线）

官方文档写了六个类名的"什么时候加、什么时候删"，但面试里能完整说出**顺序**的人不多。用 `duration` 固定时长实跑一遍，时间线是这样的（单位 ms，从切换那一刻算起）：

| 时刻 | 动作 | 元素 class |
| --- | --- | --- |
| 4 | `onBeforeLeave` 触发 | `""`（**钩子跑在加类名之前**） |
| 5 | `onLeave` 触发 | `fade-leave-from fade-leave-active` |
| 41 | 下一帧（双 `requestAnimationFrame`） | `fade-leave-active fade-leave-to` |
| 246 | `onAfterLeave`，元素已移出 DOM | `""` |
| 408 | `onBeforeEnter` 触发 | `""`（**元素还没插入 DOM**） |
| 410 | `onEnter` 触发，元素已插入 | `fade-enter-from fade-enter-active` |
| 447 | 下一帧 | `fade-enter-active fade-enter-to` |
| 650 | `onAfterEnter`，类名清空 | `""` |

从源码能读出四条硬规则：

1. **`onBeforeEnter` / `onBeforeLeave` 在加类名之前执行**。所以在这两个钩子里读 `el.className` 是读不到 `-from` 的（实测就是空串）。
2. **起始类名在插入 DOM 之前就加好了**（`onBeforeEnter` 时元素还没进 DOM），这样浏览器首次绘制时它就已经是"初始状态"，不会闪一下。
3. **`-from` → `-to` 的切换发生在下一帧**，实现是 `nextFrame = requestAnimationFrame(() => requestAnimationFrame(cb))`——**两次 rAF**。这是为了保证"初始状态已经被浏览器渲染过一帧"，否则同一次样式计算里改两次 class，浏览器只会算最终值，动画就没了。
4. **结束时三种类名一起清掉**（`-from` / `-to` / `-active` 全清），所以不用自己写清理逻辑。

由此也能解释那个经典坑：`v-leave-active` 阶段元素还在文档流里，写 `transition: all .3s` 会让它挤压兄弟节点。标准做法是给 `-leave-active` 加 `position: absolute`，把它从流里摘出去。

```css
/* 列表进出场的经典三件套 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
/* 只给 leave-active 加，保证离场元素不占位 */
.fade-leave-active {
  position: absolute;
}
```

### 7. Vue 怎么知道动画结束了：嗅探 + 兜底定时器

这是 `<Transition>` 最有意思的一环。Vue 不问你要时长，它自己"闻"：

```js
// runtime-dom：读计算样式里的 transition/animation 时长
function getTransitionInfo(el, expectedType) {
  const styles = window.getComputedStyle(el)
  const transitionTimeout = getTimeout(transitionDelays, transitionDurations)
  const animationTimeout = getTimeout(animationDelays, animationDurations)
  // 谁长听谁的；决定监听 transitionend 还是 animationend
  timeout = Math.max(transitionTimeout, animationTimeout)
  type = timeout > 0 ? (transitionTimeout > animationTimeout ? TRANSITION : ANIMATION) : null
  propCount = type === TRANSITION ? transitionDurations.length : animationDurations.length
}
```

拿到 `type` 和 `timeout` 后做三件事：

```js
// runtime-dom 的 whenTransitionEnds（简化：省略了 onEnd/end/ended 的定义）
function whenTransitionEnds(el, type, explicitTimeout, propCount, resolve) {
  if (explicitTimeout != null) return setTimeout(resolve, explicitTimeout) // 你显式传了 duration
  if (!type) return resolve()                     // 没有任何 CSS 过渡 → 直接结束，不动画
  el.addEventListener(type + 'end', onEnd)        // 正常路径：监听 transitionend / animationend
  setTimeout(() => { if (ended < propCount) end() }, timeout + 1)   // 兜底：事件没来也别卡死
}
```

三条结论：

- **没有 CSS 过渡、也没给 JS 钩子时，Vue 直接在浏览器下一个动画帧完成插入/移除**，不会白等（官方特意强调这是 rAF，不是 Vue 的 `nextTick`）。所以"我不写 CSS 就没动画"是对的，但"什么都不写会卡住"是错的。
- **有 `setTimeout(timeout + 1)` 兜底**。所以就算 `transitionend` 被浏览器优化掉（比如标签页切到后台）、或者过渡属性中途被打断，元素最终也会被正确移除。
- 想强制指定时长，`<Transition :duration="300">` 或 `:duration="{ enter: 300, leave: 500 }"` 会直接走 `setTimeout`，跳过嗅探。

### 8. JS 钩子与 done：一个参数、两个参数，行为完全不同

这是最容易写出 bug 的地方。分界线是**你的钩子函数声明了几个参数**（源码里叫 `hasExplicitCallback`）：

```js
// ✅ 只声明一个参数：Vue 仍然按 CSS 时长自动收尾
const auto = { onEnter: (el) => { el.style.opacity = 1 } }

// ⚠️ 声明了两个参数：Vue 认为"时长归你管"，不调 done 就永远不结束
const manual = {
  onEnter: (el, done) => {
    setTimeout(() => {
      el.style.opacity = 1
      done()        // 必须手动调，否则 onAfterEnter 不触发
    }, 600)
  }
}
```

实测确认：写 `onEnter(el, done)` 后把 `done()` 延到 600ms，`onAfterEnter` 就在 600ms 后才触发（而 CSS 时长只有 150ms）；写 `onEnter(el)` 一个参数时，Vue 按 `duration` 自动在 200ms 收尾。

**记这一条就够了：需要自己控制时长，才写第二个参数 `done`，并且记得调它。** 另外记得处理取消——元素在动画中途被再次切换时，`onEnterCancelled` / `onLeaveCancelled` 会触发，里面可以清掉你自己起的定时器。

### 9. mode、appear 与 TransitionGroup 的 FLIP

**`mode="out-in"`**：默认情况下新元素进场和旧元素退场是同时跑的，`out-in` 会强制"先走完再进"。实测切换时 `onBeforeLeave(a)` 在 713ms，`onBeforeEnter(b)` 和 `onAfterLeave(a)` 落在同一时刻（902ms），`onAfterEnter(b)` 在 1090ms——也就是 **a 的 leave 完全结束后才轮到 b 的 enter**（902-713 ≈ 189ms，是 150ms 时长加两次 rAF 的开销）。实现上是 `state.isLeaving` 标记加一个 `delayLeave`，把进入的时机挂到离开结束之后。

**`appear`**：默认 `false`，首次渲染**不走** enter（也就没有进场动画）。传 `appear` 后首次渲染会走 appear 系列钩子——注意这时调用的不是 `onEnter` 而是 `onAppear`，类名也换成 `appearFromClass` 等（默认跟 enter 共用同一套值）。实测 `appear: true` 时触发的是 `onBeforeAppear`，不传时两个钩子都不触发。

**`<TransitionGroup>` 的 FLIP**，源码流程分五步：

```js
// runtime-dom 的 TransitionGroup 实现（简化）
onUpdated(() => {
  const moveClass = props.moveClass || `${props.name || 'v'}-move`
  if (!hasCSSTransform(prevChildren[0].el, instance.vnode.el, moveClass)) return // ① 没配 move 过渡就跳过

  prevChildren.forEach(callPendingCbs)       // ② 把上一轮没跑完的动画先收尾
  prevChildren.forEach(recordPosition)       // ③ 记录每个元素的新位置
  const moved = prevChildren.filter(applyTranslation)  // ④ 算 dx/dy，瞬时"挪"回旧位置
  forceReflow(instance.vnode.el)             // 强制读一次布局，让上一步生效
  moved.forEach(c => {                       // ⑤ 加 moveClass，撤掉 transform → 平滑滑到新位置
    addTransitionClass(c.el, moveClass)
    c.el.style.transform = c.el.style.transitionDuration = ''
    // ... transitionend 后移除 moveClass
  })
})
```

`applyTranslation` 里的关键动作是设 `transitionDuration = '0s'` 再把元素 `translate` 回旧位置——**先瞬移回去、再撤掉瞬移**，浏览器就会把它从旧位置平滑地"过渡"到新位置。这就是 FLIP（First-Last-Invert-Play）在 Vue 里的落地。

两个实用细节：

- `<TransitionGroup>` 的子元素**必须有 key**，否则警告 `<TransitionGroup> children must be keyed.`——因为 FLIP 要靠 key 认出"同一个元素"。
- `moveClass` 默认是 `<name>-move`，而它**必须带 `transition: transform`**，否则 `hasCSSTransform` 探测不到就直接跳过整个 FLIP（这是性能优化，不是 bug）。

## 其实你每天都在用

- 表单组件里 `defineModel()` 一行搞定双向绑定——省掉的正是 `defineProps` + `defineEmits` + `computed` 那三块样板。
- 弹窗组件的 `v-model:visible`：父组件控制开，弹窗内部点关闭时 `emit('update:visible', false)`，这就是 `v-model:xxx` 的标准用法。
- 输入框加 `.trim` 处理用户从 Excel 粘过来的首尾空格，是最省事的写法，不用自己写 `@blur`。
- 搜索框加 `.lazy` 换成 `change` 触发，能避免每敲一个字符就发一次请求。
- 数字输入框写 `v-model.number`，但拿到 `''` 时不是 `0`——不做额外判断，后端就会收到空字符串。
- 多选列表用 `<TransitionGroup name="list">` 配 `:key`，增删项就有平滑位移动画，`<Transition>` 做不到（它只支持单子节点）。
- 路由切换动画 `<router-view v-slot="{ Component }">` + `<Transition mode="out-in">`，避免两个页面同时出现。
- 列表项离场时"塌陷"得很丑？八成是漏了给 `-leave-active` 加 `position: absolute`。
- 折叠面板用 `<Transition>` 做高度动画，通常得配 `onEnter(el, done)` 读一次 `scrollHeight` 再手动 `done()`——这时就踩到第 8 节那个"两个参数"的规则了。

## 常见误解（FAQ）

**❌ 误区1："`v-model` 编译成 `:value` + `@input`"**

这是 Vue 2 的答案。Vue 3 实测产物是 `onUpdate:modelValue` 回调 **+ `vModelText` 指令**，`value` 根本不在 props 里，而是指令 `mounted` 时直接写 `el.value`。而且 checkbox / radio 绑的是 `checked`、select 读的是 `options` 里被选中的项，统一说成 `:value + @input` 在四种元素里错了三种。**面试想加分，就把"不同元素用不同指令"说出来。**

**❌ 误区2："`v-model.number` 会把值转成数字，转不了就是 `NaN`"**

两个都错。实测 `'abc'` 得到的是**原字符串 `'abc'`**（转不动就原样返回，不是 `NaN`），`''` 得到的是**空字符串**而不是 `0`。更阴的是 `'12abc'` 会变成 `12`——因为它走的是 `parseFloat` 而不是 `Number()`。所以"用户输入了非法内容"这件事，`.number` 帮不了你，该校验还得校验。

**❌ 误区3："组件上用 `v-model`，子组件不在 `emits` 里声明 `update:modelValue` 会报警告"**

实测**没有警告**。源码里的告警条件是"事件既不在 `emits` 里、也没有对应的 `onXxx` prop"，而 `v-model` 会把 `onUpdate:modelValue` 当 prop 传下来，条件不成立。但**依然应该声明**：声明了它才会从 `$attrs` 里被消费掉（实测未声明时 `$attrs` 里有这个键），不声明就可能顺带 fallthrough 到根元素上。

**❌ 误区4："`<Transition>` 是个语法糖/directive"**

它是一个**函数式组件**（`(props, slots) => h(BaseTransition, resolveTransitionProps(props), slots)`），根本不是糖。它生效的机制是把自己的钩子写进子 `vnode.transition` 字段，渲染器 patch 时看到就调一下。理解这点才能解释为什么它必须单子节点、为什么不能用在列表上（要用 `<TransitionGroup>`）。

**❌ 误区5："`v-enter-from` 在元素插入之后才加"**

刚好相反：`-from` 和 `-active` 在**元素插入 DOM 之前**就加好了（实测 `onBeforeEnter` 时元素还没有 `parentNode`，class 也还是空——因为钩子跑在加类名之前）。`-from` → `-to` 的切换则在插入后的**下一帧**，用两次 `requestAnimationFrame` 保证"初始状态被渲染过一帧"。这两条合起来才是动画不闪、也不会被浏览器合成为一个样式计算的原因。

**❌ 误区6："不写 CSS transition 的话，`<Transition>` 会一直等下去 / 元素删不掉"**

不会。嗅探到 `timeout` 为 0 时直接 `resolve()`，元素照常插入/移除，只是没有动画。就算嗅探到有时长，也有 `setTimeout(timeout + 1)` 兜底——**`transitionend` 事件可能因为各种原因不来（后台标签页、属性被打断），Vue 不会傻等**。

**❌ 误区7："JS 钩子里调不调 `done` 都行，Vue 会按 CSS 时长兜底"**

要看你怎么写钩子。**钩子函数只声明一个参数时**，Vue 判定"你不用管时长"并自动收尾；**声明了第二个参数 `done`**，Vue 就认为时长归你负责，不调 `done()` 的话 `onAfterEnter` 永远不触发、后续排队的状态也全卡住。实测把 `done()` 延到 600ms，`onAfterEnter` 就在 600ms 才来。

**❌ 误区8："`<TransitionGroup>` 的 `move` 动画只要加个 `name` 就有了"**

前提是 `<name>-move` 这个类上**真的写了 `transition: transform`**。`hasCSSTransform` 会用 clone 元素探测 moveClass 的实际过渡效果，探测不到就 `return`，整个 FLIP 被跳过——表现是"移动动画完全没有，也不报错"。另外子元素必须带 `key`，否则连"谁是谁"都认不出来。

## 一句话总结

`v-model` 是编译期就被拆光的语法糖——原生元素上拆成 `onUpdate:modelValue` 回调加一个按元素类型选出来的指令（`vModelText` / `vModelCheckbox` / `vModelRadio` / `vModelSelect`），组件上拆成 `modelValue` + `update:modelValue` 的显式协议；`<Transition>` 则是货真价实的组件，靠往子 `vnode.transition` 上挂钩子被 patch 流程回调，类名按"插入前加 from+active → 两次 rAF 后换成 to → 结束全清"的节奏走，时长靠 `getComputedStyle` 嗅探加 `setTimeout` 兜底。
