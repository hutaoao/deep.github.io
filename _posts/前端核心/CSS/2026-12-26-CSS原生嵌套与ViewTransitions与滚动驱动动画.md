---
layout: post
title: "CSS 原生嵌套 / View Transitions / 滚动驱动动画"
date: 2026-12-26 00:00:00 +0800
categories: ["前端核心", "CSS"]
tags: [CSS嵌套, "View Transitions", 滚动驱动动画, "animation-timeline", 原生嵌套, 过渡动画]
description: >
  这三个特性的共同点是"同一件事以前必须靠 JS 或预处理器做，现在浏览器自己接管了"：嵌套接管了 Sass，View Transitions 接管了手写 FLIP，滚动驱动动画接管了 scroll 事件。面试要答的是机制和使用边界。
---

## 一句话概括

这三个特性看着不搭边，但它们解决的是同一类问题：**过去要靠构建工具或 JS 才能实现的东西，浏览器自己做了**。

1. **"组件样式想按结构缩进着写"** → 以前只能上 Sass 的嵌套 → 现在 CSS 原生嵌套
2. **"切页面 / 切 tab / 点开大图要有过渡动画"** → 以前要么手写 FLIP，要么上过渡库 → 现在 View Transitions API
3. **"滚动到某个位置就播放动画"** → 以前只能 `scroll` 事件 + `requestAnimationFrame`，跑在主线程必掉帧 → 现在滚动时间轴（scroll-driven animations）

三者的"含金量"在同一点上：**执行位置从 JS 主线程挪到了浏览器的样式/合成层**。嵌套把工作挪到了 CSS 解析期；View Transitions 把动画交给合成器（浏览器只截快照、剩下的交给 GPU）；滚动时间轴让动画脱离主线程。

先给一张版本表（数据来自 MDN 浏览器兼容数据；面试被追问版本时报得不精确很减分，但记 **Baseline 年份**更实用）：

| 特性 | Chrome | Safari | Firefox | Baseline |
| --- | --- | --- | --- | --- |
| CSS 原生嵌套（含省略 `&`） | 120 | 17.2 | 117 | 2023-12 |
| `document.startViewTransition()`（同文档） | 111 | 18 | 144 | 2025-10 |
| `@view-transition`（跨文档 MPA） | 126 | 18.2 | 尚未支持 | — |
| 滚动驱动动画（`animation-timeline`） | 115 | 26 | 尚未支持（在 flag 后面） | — |

## 核心知识点

### 1. 原生嵌套的第一个坑：`&` 不是字符串，是选择器引用

很多人第一次用原生嵌套，脑子里装的是 Sass 的 `&`——**Sass 的 `&` 是文本替换，原生 CSS 的 `&` 是"父选择器的引用"**。差别只有一个，但后果很大：

```css
/* ❌ 你以为在拼 BEM 类名，其实不是 */
.card {
  &__title { font-weight: 700; }   /* 不产生 .card__title */
}

/* ✅ 老老实实写全类名 */
.card { }
.card__title { font-weight: 700; }
```

原因：`&__title` 被解析成"父选择器 + 一个叫 `__title` 的类型选择器"这样的复合选择器，页面上根本没有 `<__title>` 这个标签，所以什么也匹配不到。本机实测：给一个 `<div class="cn__t">` 同时挂 `.cn__t { background: rgb(9,9,9) }` 和 `.cn { &__t { background: rgb(1,2,3) } }`，最终颜色是 `rgb(9,9,9)`——`&__t` 那条**一条都没生效**，而且是**静默失效，控制台不报错**。

**记住这一条就够了：原生嵌套里 `&` 只能和自己的"分隔符修饰符"（`&:hover`、`&.active`）拼，不能拼出新的标识符（名字）——`&__title`、`&--mod` 这类 BEM 写法是 Sass 专属能力，在原生嵌套里是无效选择器。**

### 2. 原生嵌套的第二个坑：`&` 的权重等于 `:is(父选择器列表)`

这是规范里写死的：**`&` 的权重 = 父规则选择器列表里最具体的那一个**（和 `:is()` 完全一样的算法）。

```css
#sidebar, .widget {
  & i { outline-color: rgb(1,2,3); }   /* 权重 = (1,0,1)，因为取了 #sidebar 的 ID 权重 */
}

.zz i { outline-color: rgb(9,8,7); }    /* 权重 = (0,1,1) */
```

本机实测：一个 `class="widget"` 的元素，它的 `<i>` 拿到的是 `rgb(1,2,3)`——`.widget` 明明没有 ID，但 `&` 还是按 `#sidebar` 的权重算了。

**面试可以这么说**：因为如果按"实际匹配到哪个父选择器"决定权重，浏览器就得为同一个选择器维护多个权重，选择器匹配会变得极慢（规范原文给了这个理由），所以 `&` 直接取"最大权重"。**后果是：只要父规则的选择器列表里混进一个 ID 选择器，里面所有嵌套规则的权重都会被拉高**，你要覆盖它就得跟着写 ID。工程建议：父规则里别混 ID 选择器。

### 3. 原生嵌套的第三个坑：省略 `&` 的规则，语义会变

原生嵌套允许省略 `&`（这叫 relaxed parsing），但有两条完全不同的规则，很多人混着记：

```css
/* ✅ 元素选择器可以省略 &，等于后代选择器 */
.card {
  h2 { font-size: 1rem; }   /* 等于 .card h2 */
}

/* ❌ 伪类不能省略 &！省略了就变成"后代里的伪类" */
.card {
  :hover { color: red; }    /* 等于 .card *:hover —— 鼠标移到任意后代都变色 */
  &:hover { color: red; }   /* ✅ 这才是 .card:hover */
}
```

本机实测第二种：写 `.wrap { :first-child { color: rgb(1,2,3) } }`，颜色落到了 `.wrap` 的**第一个子元素**身上，`.wrap` 自己没变。MDN 的原文说得很清楚：省略 `&` 时浏览器会自动在中间**加一个空格**，于是变成后代选择器。

**版本边界要记住**：Chrome 112~119、Safari 16.5~17.1 只支持"带 `&`"的写法，**不支持以类型选择器开头的嵌套规则**——也就是说 `h2 { }` 这种省略写法在老版本上会**整条规则被丢弃且不报错**（MDN 兼容数据明确标了 `Does not support nested rules that start with a type selector`）。真正安全的分界线是 **Chrome 120 / Safari 17.2 / Firefox 117**。

### 4. 声明和嵌套规则混写：别赌，把声明放前面

```css
/* 规范提醒过这点，各家实现历史上有过差异 */
.card {
  color: rgb(1,2,3);
  & { color: rgb(9,9,9); }
  color: rgb(5,5,5);     /* 这条会被"提到"嵌套规则前面吗？ */
}
```

规范里有一条 Note：序列化时**直接嵌套的声明会被提到嵌套规则之前**（所以官方建议把声明写在嵌套规则前面）。但本机在 Chrome 154 实测，`color` 最终是 `rgb(5,5,5)`——**写在嵌套规则之后的声明真的赢过了嵌套规则**，也就是说实际参与层叠的顺序跟源码顺序一致。查 CSSOM 也能看到浏览器保留的是源码顺序。

结论不用纠结谁对：**声明统一写在嵌套规则之前**，任何实现、任何版本下结果都一样；反过来写就是在赌实现细节。

### 5. 原生嵌套不等于 Sass，能力边界要能说清

| 能力 | Sass / Less | 原生嵌套 |
| --- | --- | --- |
| 基础嵌套（后代） | ✅ | ✅ |
| `&:hover` / `&.modifier` | ✅ | ✅ |
| 嵌套 `@media` / `@container` | ✅ | ✅ |
| `&__elem` 字符串拼接（BEM） | ✅ | ❌（静默失效） |
| 变量 | ✅ 编译期 | ❌ 用 CSS 变量代替 |
| mixin / 函数 / 循环 | ✅ | ❌ |
| 输出 | 编译成扁平 CSS | 浏览器自己解析 |
| 需要的浏览器版本 | 无（产物是普通 CSS） | Chrome 120 / Safari 17.2 / Firefox 117 |

所以"原生嵌套能替掉 Sass"只说对了一半：**它替掉的是"为了嵌套而引入 Sass"这个需求**。真要兼容老浏览器，构建链路里的 Lightning CSS / PostCSS 嵌套插件还得留着——它们会把嵌套**编译成扁平选择器**，产物对浏览器没有要求。本机用 lightningcss 实测产物：

```css
/* 输入 */
.card { padding: 1rem; &:hover { color: red; } .title { font-weight: 700; } }

/* lightningcss 指定 chrome 100 后的输出：全部拍平 */
.card { padding: 1rem; }
.card:hover { color: red; }
.card .title { font-weight: 700; }
```

### 6. View Transitions（同文档）：一次 DOM 变更的前后快照动画

原理一句话：**浏览器给"改 DOM 前"和"改 DOM 后"各拍一张快照，然后用伪元素把两张快照交叉淡出淡入，真实 DOM 早就已经是最终状态了。**

```js
// 把"引起 DOM 变化"的操作包进回调里
document.startViewTransition(() => {
  list.replaceChildren(...newItems);   // 同步改 DOM
});
```

整个过程分六步（MDN 原文顺序）：

1. 触发过渡
2. 在老视图上**截快照**——只截有 `view-transition-name` 的元素，其余归 `root` 快照
3. 执行回调，DOM 变成新状态（回调跑完 `updateCallbackDone` 兑现）
4. 在新视图上截快照（`ready` 兑现）
5. 老快照"淡出"、新快照"淡入"（默认 cross-fade）
6. 动画结束，快照销毁（`finished` 兑现）

两个细节特别值钱：

- **老快照是一张静态图片（不可交互），新快照是可交互的真实 DOM 区域**。这是有意的：过渡期间用户不能点到"上一屏的残影"，但可以点新内容。
- 过渡的伪元素树是**叠在页面之上（overlay）**的，所以动画本身不会引起布局位移。

```text
::view-transition
└─ ::view-transition-group(root)
   └─ ::view-transition-image-pair(root)
      ├─ ::view-transition-old(root)   ← 老快照（静态图片）
      └─ ::view-transition-new(root)   ← 新快照（活的）
```

想改动画就覆盖这两层：

```css
/* 给"共享元素"单独起名字：列表页的封面图 -> 详情页的大图 */
.hero-image { view-transition-name: hero; }

/* 只对 hero 这一组做自定义动画，其余保持默认 cross-fade */
::view-transition-old(hero) { animation: 200ms ease-out both fade-out; }
::view-transition-new(hero) { animation: 200ms ease-out both fade-in; }
```

### 7. View Transitions 的三条硬约束

**① `view-transition-name` 必须唯一。** MDN 原文：同一个时刻有两个渲染中的元素用了同一个名字，`ViewTransition.ready` 这个 Promise 会 **reject，并且整个过渡被跳过**。"唯一"的范围是**参与过渡的那一屏**（不要给列表里 100 个卡片都写同一个名字）。列表场景用 Level 2 的 `view-transition-name: match-element`，浏览器会给每个元素自动分配唯一内部名字——但它**只能用于同文档过渡**，跨页面不可用。

**② 必须在 `prefers-reduced-motion` 的开关里。** 自定义动画（尤其是位移、缩放这类"动得多"的）一定要包起来：

```css
@media (prefers-reduced-motion: no-preference) {
  ::view-transition-old(root) { animation: 200ms ease-in both slide-out; }
  ::view-transition-new(root) { animation: 200ms ease-out both slide-in; }
}
```

**③ 页面不可见时，过渡会被整个跳过。** MDN 明确写了：调用 `startViewTransition()` 时如果文档的页面可见性状态是 `hidden`（窗口被遮住、浏览器最小化、标签页切走了），过渡直接跳过——所以**别把"过渡结束"当成业务逻辑的时序依赖**（要等结果就 `await transition.finished`，但要接受它可能被跳过）。

顺带：**这个 API 必须自己特性检测**，不支持时它压根不存在，直接调用会抛 TypeError：

```js
// ✅ 优雅降级：没有动画，DOM 该改还是改
function navigate(updateDOM) {
  if (!document.startViewTransition) { updateDOM(); return; }
  document.startViewTransition(() => updateDOM());
}
```

### 8. 跨文档 View Transitions（MPA）：一行 CSS，不用 JS

这是最容易被忽略、也最"划算"的一块：**多页应用（普通 `<a>` 跳转、服务端渲染的站点）也能有过渡动画，而且不需要路由、不需要 JavaScript。**

```css
/* 两个页面（出发页 + 目标页）都要有这一行 */
@view-transition { navigation: auto; }
```

加了它，**同源**的普通导航就会自动获得 cross-fade；外链、刷新都不受影响。`navigation` 还支持 `none`，用来给某些路由单独关掉。

两个坑：

- **这条 CSS 必须在出发页首次绘制前就被解析到**。如果它是懒加载进来的，下一次导航的过渡会被**静默跳过**（没有报错，就是没动画）——所以放关键 CSS 里，别放异步样式表。
- 想在这两个时机做定制，用 `pageswap`（老页面，导航离开时）和 `pagereveal`（新页面，揭示时）两个事件。共享元素的做法不变：**两个页面对应元素起同一个 `view-transition-name`**。

### 9. 滚动驱动动画：把"滚动位置"变成一根时间轴

核心心智模型只有一句：**以前动画的时间轴是"时钟"，现在可以换成"滚动条的位置"。**

关键原因不是语法好看，是性能：Chrome 官方博文写得很直接——传统做法要监听 `scroll` 事件在主线程里算，而**现代浏览器的滚动跑在独立进程、事件是异步派发的，主线程动画必然抖动（jank）**。滚动时间轴让这类动画**可以完全不占主线程**。

两种时间轴，对应两类需求：

| 时间轴 | 谁在推进 | 典型场景 | 写法 |
| --- | --- | --- | --- |
| **滚动进度时间轴**（Scroll Progress Timeline） | 滚动容器的**滚动位置**（0% 到 100%） | 阅读进度条、视差背景 | `animation-timeline: scroll(root)` |
| **视图进度时间轴**（View Progress Timeline） | 某个元素在滚动容器里的**可见进度** | 图片进入视口淡入、元素跑到中间时放大 | `animation-timeline: view()` |

```css
/* 阅读进度条：整个页面滚到底 = 动画跑完 */
.progress {
  animation: grow linear;
  animation-timeline: scroll(root);   /* root | nearest | self */
}

/* 卡片进入视口时淡入 */
@keyframes reveal { from { opacity: 0; translate: 0 40px; } }

.card {
  animation: reveal linear;
  animation-timeline: view();         /* 等价于 view(block auto) */
}
```

`scroll()` 的参数是 `<scroller>`（`root` / `nearest` / `self`）和 `<axis>`（`block` / `inline` / `x` / `y`），不写是 `scroll(nearest block)`。`view()` 跟它不一样的地方是：**它不需要指定滚动容器**——按定义就是"元素在最近的祖先滚动容器里"的可见进度。

### 10. `animation-timeline` 的三个使用陷阱（都要背）

**陷阱一：`animation-timeline` 是"仅重置"属性，写在 `animation` 简写后面才有效。**

规范里 `animation` 这个简写把 `animation-timeline` 列为 **reset-only** 分量：写 `animation: ...` 会把它重置回 `auto`，而且**你不能在 `animation` 简写里给它赋值**。所以顺序错了就白写：

```css
/* ❌ 被简写重置成 auto：动画跑回"按时间播放" */
.foo { animation-timeline: scroll(root); animation: grow 2s linear; }

/* ✅ 简写在前、时间轴在后 */
.bar { animation: grow 2s linear; animation-timeline: scroll(root); }
```

本机实测（Chrome 154）：前者 `getComputedStyle` 拿到 `animation-timeline: auto`，动画对象的 `timeline` 是 `DocumentTimeline`；后者拿到 `scroll(root)`，`timeline` 是 `ScrollTimeline`，`playState` 为 `running`。**这是最容易"代码看着没错但动画不对"的一处。**

**陷阱二：`animation-duration` 对滚动时间轴基本没意义，但别不写。**

MDN 的 Note 说得很实在：对滚动进度时间轴，`animation-duration` 设多少秒**测试下来看不出效果**；对视图进度时间轴，它会把动画"推到更靠后"；但 **Firefox 要求必须设了 `animation-duration` 才应用动画**，所以建议统一写 `1ms`——能生效，又不至于改变效果。另外 `animation-duration: auto` 在滚动时间轴下的含义是"铺满整根时间轴"。

**陷阱三：不支持时不是"不动"，而是"按时间自动播一遍"。**

这条最坑：在不支持 `animation-timeline` 的浏览器里，`animation` 本身是支持的，于是那几行 `@keyframes` 会退化成一个**普通的时间动画**——用户还没滚到，动画已经播完了。所以滚动动画**必须用 `@supports` 隔离**：

```css
@supports (animation-timeline: scroll()) {
  .progress { animation: grow linear; animation-timeline: scroll(root); }
}
```

### 11. `animation-range`：动画在时间轴的哪一段发生

默认情况下，视图时间轴是"元素刚露头 = 0%，元素完全离开 = 100%"。这往往不是你要的——元素刚进入视口时动画已经跑了一大截。

`animation-range` 就是用来"裁时间轴"的。它取 `<start> <end>` 两个位置，位置可以是百分比，也可以用**具名区间**：`cover`（默认，含整个滚动过程）、`contain`、`entry`、`exit`、`entry-crossing`、`exit-crossing`。MDN 明确提醒：因为 100% 通常发生在元素离开视口那一刻，所以**通常应该把动画结束写在 20% / 50% / 80% 这种键帧上，而不是 `to`**，否则元素都看不见了动画还没播完。

```css
.card {
  animation: reveal linear;
  animation-timeline: view();
  animation-range: entry 0% cover 50%;   /* 本机实测计算值就是 entry cover 50% */
}
```

`animation-range` 的初始值是 `normal`（本机实测计算值就是 `normal`）。`normal` 在视图时间轴下等价于 **`cover 0% cover 100%`**（在滚动时间轴下等价于 `0% 100%`）——也就是说**默认动画铺满"元素从开始露头到完全离开"的整段**，这就是"元素刚露头动画就开播、离开才播完"的来源。

还有一个进阶件：`timeline-scope`。**命名时间轴默认只对"声明它的元素的后代"可见**，如果你想让**不在这个子树里的元素**（比如兄弟节点、外层兄弟区域）也能引用同一根时间轴，就在它们的**共同祖先**上写 `timeline-scope: --my-timeline;`，把这个名字提升到那个作用域。面试提到这个基本就够用了。

## 其实你每天都在用

- **Vue 的 `<Transition>` / React 的 `AnimatePresence`**：它们本质上是手写的"进/出快照 + FLIP"，View Transitions 把这件事下沉到了浏览器。
- **你在 `.scss` 里写的 `&:hover`**：语法和原生嵌套几乎一样，但一个是编译期文本替换，一个是浏览器解析——`&__title` 就是两者分家的地方。
- **Google Photos / Chrome 标签页里"点缩略图放大成整屏"的动效**：这就是 View Transitions 最典型的产品形态（共享元素 morph）。
- **博客阅读进度条、页面顶部的"回到顶部"按钮显隐**：以前要 `scroll` + `rAF` 节流，现在几行 CSS。
- **产品页"图片滚动到中间时放大"的视差效果**：`animation-timeline: view()` 的经典案例。
- **首页那种"滚到才淡入"的卡片列表**：以前靠 `IntersectionObserver` 加类名，现在用 `view()` 时间轴的 `entry` 区间，还不用管"顺序触发"的定时器。
- **Chrome DevTools 的 Animations 面板**：能直接看到滚动驱动动画的进度条（带滚动图标），调试时一眼分清"时间轴动画"和"滚动动画"。
- **你项目里那个"切换 tab 时有闪一下"的组件**：`startViewTransition` 一行就能修，而且不引入任何动画库。

## 常见误解（FAQ）

**❌ 误区一："原生嵌套出来了，Sass 可以删了"**

分场景。原生嵌套**替代不了** Sass 的 `&__elem` 拼接、mixin、`@each` 循环和编译期变量（这些是"生成能力"）；它也**不能给老浏览器兜底**（Chrome 120 / Safari 17.2 / Firefox 117 以下不认，而且不认的时候规则是被**整条丢弃、不报错**）。所以：只需要嵌套、且浏览器基线够新 → 可以删 Sass；需要 BEM 拼接、需要兼容老浏览器、或者本来就在用 CSS Modules/PostCSS 链路 → 留下构建期的嵌套编译更稳。

**❌ 误区二："嵌套里的 `&` 就是父选择器的字符串"**

它是**选择器引用**，不是字符串。两个直接后果：`&__title` 不会拼出 `.card__title`（而是解析成"父选择器 + `__title` 类型选择器"，匹配不到任何东西，**静默失效**）；`&` 的权重按 `:is(父选择器列表)` 取**最大值**（本机实测父列表里有 `#id` 时，`.class` 分支匹配到的元素也拿到 ID 级权重）。

**❌ 误区三："省略 `&` 和写 `&` 是一回事"**

只有**类型选择器**可以省略（`.card { h2 { } }` 等价 `.card h2`）；**伪类不能省略**——`.card { :hover { } }` 被解析成 `.card *:hover`，鼠标移到任何后代上都会触发。本机实测 `.wrap { :first-child { } }` 命中的是 `.wrap` 的第一个子元素，不是 `.wrap` 自己。而且省略写法有版本边界（Chrome 120 / Safari 17.2 之前不支持以类型选择器开头的嵌套规则）。

**❌ 误区四："View Transitions 只能用于 SPA"**

同文档过渡（`document.startViewTransition`）确实要 JS，但**跨文档过渡是纯 CSS**：两个页面都写上 `@view-transition { navigation: auto; }` 就行，普通 `<a>` 导航就有动画，不需要路由、不需要框架（Chrome 126+ / Safari 18.2+，Firefox 尚未支持）。前提是同源导航，而且这条 CSS **必须在首次绘制前解析到**，否则静默跳过。

**❌ 误区五："不支持 View Transitions 的浏览器会白屏 / 报错"**

不会白屏，但**也不会有优雅降级**——`document.startViewTransition` 在不支持的浏览器里根本不存在，直接调会抛 `TypeError`。所以必须自己检测：不支持时直接执行 DOM 更新，跳过动画。跨文档那套反而天生安全，因为不支持时 `@view-transition` 就是一条被忽略的规则，导航照常发生。

**❌ 误区六："滚动动画还是得监听 scroll 事件"**

原生滚动时间轴的关键优势就是**不跑主线程**（Chrome 官方博文的原话是让滚动动画 off the main thread）。传统的 `scroll` + `rAF` 之所以抖，是因为滚动已经被浏览器放到独立进程、事件异步送达，你在主线程做的动画永远追不上滚动位置。真要做更复杂的联动（比如根据滚动数据改业务状态），`IntersectionObserver` 仍然比 `scroll` 事件划算。

**❌ 误区七："`animation-timeline` 和 `animation` 谁先写都行"**

不行。`animation-timeline` 在 `animation` 简写里是 **reset-only** 分量——写 `animation: ...` 会把它重置成 `auto`，而且你没法在简写里给它赋值。本机实测：写在简写前面 → 计算值 `auto`、动画挂在 `DocumentTimeline` 上；写在后面 → 计算值 `scroll(root)`、挂在 `ScrollTimeline` 上。**顺序错了动画会"变成按时间播一遍"，但不报错。**

**❌ 误区八："不支持滚动时间轴的浏览器里，动画就是不动，挺安全"**

反了。`animation` 本身是支持的，所以那套 `@keyframes` 会退化成一个**普通时间动画**——用户还没滚到，动画已经自动播完了，视觉上就是"该出现的元素莫名其妙已经出现过"。滚动驱动动画必须包在 `@supports (animation-timeline: scroll())` 里。

## 一句话总结

**这三个特性的共同方向是把过去由 JS 和预处理器承担的活交给浏览器：原生嵌套用 `&`（选择器引用、权重取 `:is()` 最大值、`&__elem` 拼接失效）替代 Sass 的嵌套；View Transitions 用"前后快照 + 伪元素树"替代手写 FLIP，同文档靠 `startViewTransition()`、跨文档靠一行 `@view-transition`；滚动驱动动画用 `scroll()` / `view()` 时间轴替代 `scroll` 事件 + rAF，从而脱离主线程——但它要写在 `animation` 简写之后，且必须用 `@supports` 隔离，否则会退化成"按时间自动播一遍"。**
