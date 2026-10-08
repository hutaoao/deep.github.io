---
layout: post
title: "现代 CSS 新特性：容器查询 / 子网格 / :has() / @layer"
date: 2026-12-24 00:00:00 +0800
categories: ["前端核心", "CSS"]
tags: [容器查询, 子网格, ":has()", "@layer", 响应式布局, CSS新特性]
description: >
  这四个特性解决的其实是同一件事：CSS 一直缺"组件级响应式""跨层级继承""向上选择"和"可控的层叠顺序"。面试要答的是各自的机制与代价，不是背语法。
---

## 一句话概括

现代 CSS 有四个"补齐短板"的新特性，面试问它们的原因很集中——**以前的 CSS 有四个做不到的事**：

1. **元素只能按视口适配，不能按自己容器的大小适配** → 容器查询（`@container`）
2. **嵌套 grid 无法和父 grid 对齐** → 子网格（`subgrid`）
3. **选择器只能向下选，不能选父元素** → `:has()`
4. **层叠冲突只能靠权重和源码顺序硬拼** → `@layer`

也就是说，这四个不是一个"新语法合集"，而是四个问题的解。面试官真正想看的是你知不知道**每个解法的代价**：容器查询会带来包含（containment）副作用、`:has()` 有性能成本、`@layer` 会打乱你对"谁压谁"的直觉、子网格在自动排布上有限制。

先给一张兼容表（数据来自 MDN 浏览器兼容数据，面试报版本号会被追问，记 Baseline 年份更实用）：

| 特性 | Chrome | Safari | Firefox | Baseline |
| --- | --- | --- | --- | --- |
| `@layer` | 99 | 15.4 | 97 | 2022-03 |
| 容器尺寸查询 | 105 | 16 | 110 | 2023-02 |
| 子网格 | 117 | 16 | 71 | 2023-09 |
| `:has()` | 105 | 15.4 | 121 | 2023-12 |
| `@container` 样式查询 | 111 | 18 | 尚未支持 | — |
| `@container` 滚动状态查询 | 133 | 尚未支持 | 尚未支持 | — |

## 核心知识点

### 1. 容器查询：查询对象从"视口"变成"祖先容器"

一句话：**媒体查询问"窗口多大"，容器查询问"我这个组件被塞进了多大的盒子"**。

这解决了一个老问题：同一个卡片组件放进宽主栏和窄侧栏，用媒体查询它长得一模一样（因为它只知道窗口宽度），所以以前只能靠 JS 的 `ResizeObserver` 补丁，或者给组件加一堆 `--compact` 之类的修饰类。

机制是两步，**两步缺一不可**：

```css
/* 第一步：祖先声明自己是查询容器 */
.card-wrap {
  container-type: inline-size;   /* 只查 inline 轴（横排就是宽） */
  container-name: card;          /* 可选：给它起个名字 */
}

/* 第二步：后代写 @container，条件写法跟 @media 一样 */
@container card (min-width: 400px) {
  .card { display: flex; flex-direction: row; }
}
```

`container-type` 只有三个常用值，区别在"能查什么"和"包含什么"：

| 值 | 能查什么 | 附带什么 |
| --- | --- | --- |
| `normal`（默认） | 只能做样式查询、仅名字查询 | 无 |
| `inline-size` | 只能查 inline 轴（宽度） | style + inline-size 包含 |
| `size` | 能查宽和高两个轴 | style + **size** 包含（两个轴都锁） |

工程里 **95% 的情况用 `inline-size`**，因为高度照旧由内容撑开，不会破坏"高度跟着内容走"的布局习惯。用 `size` 就得自己保证高度有明确来源，否则塌陷（下面第 4 节实测）。

### 2. 容器查询的三条硬规则（面试最爱追问）

**规则一：元素不能查自己，只能查祖先。**

```css
/* ❌ 自问自答：这个规则永远不生效 */
.box { container-type: inline-size; }
@container (min-width: 300px) { .box { color: red; } }

/* ✅ 包一层：外层当容器，内层当被查询者 */
.box-wrap { container-type: inline-size; }
@container (min-width: 300px) { .box { color: red; } }
```

本机实测：上面 ❌ 写法里 `.box` 的颜色不变。这不是 bug，是防止循环依赖——如果元素能根据自己尺寸改样式，改样式又可能改尺寸，就会无限循环。

顺带说：**根本没有祖先容器时，规则也不匹配**（实测：`@container (min-width: 1px)` 在没有容器的元素上不生效）。所以排查"容器查询没生效"先看这两条：容器挂在哪、有没有容器。

**规则二：命名查的容器是"最近的那个**有这个名字**的祖先，不是最近的容器。**

这是命名真正的作用——**防止组件的查询被使用方外层容器劫持**。本机实测很有代表性：

```
.outer  { container-type: inline-size; container-name: oc; width: 900px }
.inner  { container-type: inline-size; width: 300px }   /* 没名字 */
.deep   { ... }

@container oc (min-width: 800px) { .deep { color: red } }   /* 命中，因为 inner 没有这个名字 */
```

`.deep` 最近的容器是 300px 的 `.inner`，但 `.inner` 没有 `oc` 这个名字，所以 `@container oc (...)` 直接跳到 900px 的 `.outer`，条件成立。**给组件内部用的容器起名字，是让它不受使用方布局影响的唯一办法。**

**规则三：`inline-size` 容器查不了高度。**

```css
.iq  { container-type: inline-size; height: 300px; }
@container (min-height: 100px) { .target { color: green } }   /* ❌ 不生效 */
```

实测确认：换 `container-type: size` 才生效。面试问"为什么要用 `size`"，答案就是这一条——你要查高度，代价是高度也得由上下文或显式值给出。

### 3. 代价：包含（containment）会让容器塌陷

这条是最容易踩的，也是最能体现"你真的用过"的一条。

`container-type` 不是纯声明式的开关，它同时给元素加了 **containment**（包含）。原因 MDN 写得非常直白：**尺寸包含会关掉元素"从内容拿尺寸信息"的能力，这是为了避免循环**——容器查询里的规则改了内容尺寸 → 反过来可能让查询条件翻转 → 再改内容尺寸 → 无限循环。

**MDN `container-type` 原文**：

> The container size has to be set by context, such as block-level elements that stretch to the full width of their parent, or explicitly defined. **If a contextual or explicit size is not available, elements with size containment will collapse.**

本机实测（Chrome，宽度靠内容撑的元素）：

| 场景 | 普通元素宽度 | 加 `container-type: inline-size` 后 |
| --- | --- | --- |
| `display: inline-block` + 一段文字 | 148.4px | **0px** |
| `position: absolute` + 一段文字（收缩包裹） | 148.4px | **0px** |
| flex 子项 + 一段文字 | 148.4px | **0px** |

`container-type: size` 更狠，块轴也锁：一个只有文字的 `div` 加 `container-type: size` 后**高度实测为 0**。

**结论：容器要挂在"宽度本来就有来源"的元素上**——块级元素（自动撑满父宽）、grid/flex 里给定尺寸的轨道、或者自己写了 `width` 的盒子。**别挂在 `inline-block` / `float` / 绝对定位这类收缩包裹的元素上。**

（顺带澄清一个常见担心：本机实测 `container-type: inline-size/size` **不会**像 `transform` 那样把容器变成 `position: fixed` 后代的包含块，fixed 子元素依旧相对视口定位。）

### 4. `cq` 单位与另外三种容器查询

容器查询带来的长度单位，**`1cqi` = 容器 inline size 的 1%**（不是视口的 1%，横排时 `cqi` 就等于 `cqw`）：`cqw` / `cqh` / `cqi` / `cqb` / `cqmin` / `cqmax`。

本机实测：容器 500px 宽时 `font-size: 1cqw` 算出 **5px**。这就是"组件内的字号跟着组件大小缩放"的做法，比 `vw` 靠谱得多——同一个组件放主栏和侧栏，字号比例都正确。

一个容易混的细节要分清两种情况（MDN 原文只说后者）：**浏览器支持 `cq` 单位、但没有可用的查询容器时**，`cq` 单位会**回落到小视口单位 `sv*`**——实测在无容器环境里 `1cqw` 算出了 7.56px（就是此时视口宽度的 1%）。而**浏览器根本不认识 `cq` 单位时**，这条声明是无效声明、会被整体丢弃，**不会**退化成 `vw`。所以 `cq` 单位要么配一条前置的兜底声明，要么用 `@supports` 包起来：

```css
.card__title {
  font-size: 1.25rem;                       /* 兜底：不认识 cqw 的浏览器用这条 */
  font-size: clamp(1rem, 4cqi, 2rem);       /* 支持的浏览器用这条 */
}
```

容器查询目前一共四类，面试提到后两类是加分项：

- **尺寸查询**（主流）：`@container (min-width: 400px)`
- **样式查询**：`@container style(--theme: dark)`，查**自定义属性的计算值**。实测能命中；支持度是 Chrome 111+ / Safari 18+，Firefox 尚未支持
- **滚动状态查询**：查容器是否被滚动、是否 sticky 吸附、是否吸附目标。Chrome 133+ 才有
- **仅名字查询**：`@container card { ... }` 不带条件，只判断"有没有这个名字的祖先容器"。**注意**：仅名字查询不要求容器有尺寸类型，`container-type: normal` 也能当容器（实测直接命中了 `@container justname { }`）

降级写法照旧用 `@supports`：`@supports (container-type: inline-size) { ... }`（实测生效）。

### 5. 子网格：让嵌套 grid 对齐父网格的轨道

问题背景一句话：**grid 只有直接子元素能放进网格，嵌套 grid 是独立坐标系**。所以"卡片里的标题、正文要和页面的三列网格对齐"这种需求，以前只能靠算宽度、传变量、或者 JS 测量。

`subgrid` 的机制是"不新建轨道，直接用父级的轨道"：

```css
.page { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; }

/* 这个卡片横跨父网格 3 列，它自己的列轨道 = 父级的那 3 条 */
.card { grid-column: 1 / -1; display: grid; grid-template-columns: subgrid; }
```

本机实测：父网格 `grid-template-columns: repeat(3, 100px); gap: 10px`，子网格横跨 3 列后，它自己的宽度实测 **320px**（= 3×100 + 2×10 gap），而里面每个条目实测宽 **100px**——**真的在复用父级轨道，不是自己算的**。

三个值得记的细节：

1. **行号在子网格里从 1 重新开始**（组件放到页面哪个位置，内部行号写法都一样——这才是它对组件化真正友好的地方）
2. **gap 从父网格继承**，可以在子网格上覆盖
3. **线名（line names）可以从父级传进来**，子网格也能自己声明：`grid-template-columns: subgrid [line1] [line2]`

### 6. 子网格最反直觉的一条：不产生隐式轨道

这条是"用过才知道"的坑：**普通嵌套 grid 会自动生成隐式轨道（多出来的项就往下长），子网格不会。**

本机实测对照（父网格 3 列 × 2 行、`gap: 10px`，子网格跨满两行放 9 个子项）：

| | 子网格（`rows: subgrid`） | 普通嵌套 grid |
| --- | --- | --- |
| 子网格自身高度 | 90px（= 2×40 + 10px gap，与父级两行严丝合缝） | content 高 **122px**（自动多长出一行） |
| 第 7~9 个子项去哪了 | 挤在**最后一条轨道**里（与第 4~6 项位置重叠） | 落到新生成的隐式行上 |
| 父级布局受影响吗 | 完全不受影响（子网格高度被父轨道锁住） | 内容溢出父级给的那两行 |

所以：**知道要放几个、能整除时用子网格；数量不定要靠自动排布时，子网格维度就得去掉**（把 `grid-template-rows: subgrid` 换成正常值，隐式行才会恢复，但代价是与父网格行不再对齐）。

支持度上它已经是 Baseline（2023-09），Chrome 117 / Safari 16 / Firefox 71（Firefox 反而最早）。

### 7. `@layer`：把"层叠冲突"从权重博弈变成显式排序

`@layer` 的定义很克制：**它只改一件事——层叠顺序的比较位置**。CSS 层叠比顺序是这样的（从高到低）：

```
transitions > !important 声明 > 普通声明 > 动画
```

而普通声明之间的比较，正常是 **权重 → 源码顺序**。`@layer` 把"层"插到了权重**前面**：

```
层先后顺序  →  权重  →  源码顺序
```

也就是说**只要层序不同，权重再高也没用**。本机实测（Chrome）：

```css
@layer lowprior, highprior;   /* 先声明顺序：highprior 在后 → 优先级更高 */

@layer highprior { .spec { color: rgb(200, 0, 0); } }          /* 权重 0,1,0 */
@layer lowprior  { #spec-id.spec-x { color: rgb(210, 0, 0); } } /* 权重 1,1,0 */

/* 实测：.spec 赢（rgb(200,0,0)）——ID 选择器在后面的层面前毫无抵抗力 */
```

**这就是 `@layer` 的真正价值：你不再需要为了压过第三方库而写更长的选择器、或者上 `!important`。**排序问题用声明顺序解决，选择器就能一直保持简单。

官方推荐的用法是先声明顺序、再往里塞规则：

```css
/* 只声明顺序，不带规则 */
@layer reset, vendor, base, components, utilities;

/* 之后再往对应层里追加规则，层序不会变 */
@layer components { .btn { ... } }
```

### 8. `@layer` 的四条反直觉规则

这四条几乎每条都有人答错，全部本机实测过：

**① 未分层（unlayered）的样式，压过所有分层样式。**

```css
@layer base { .u { color: rgb(1,2,3); background: rgb(9,9,9); } }
.u { color: rgb(0,0,255); }   /* 未分层 */

/* 实测：color = 蓝（未分层赢）；background = rgb(9,9,9)（没人跟它抢，照样生效） */
```

MDN 原文：*"Styles that are not defined in a layer always override styles declared in named and anonymous layers."* 换句话说，**你不写 `@layer` 的样式永远在最上层**——这既是"引第三方库时我的代码天然能压它"的方便，也是"我想把老代码整体降级"的障碍。

**② 重新声明层名只追加规则，不改顺序。**

```css
@layer first, second;
@layer second { .o { color: rgb(50,0,0); } }
@layer first  { .o { color: rgb(60,0,0); } }
/* 实测：rgb(50,0,0) —— 顺序由第一次声明锁定 */
```

**③ `!important` 反转层序，并且重要声明内部再排一层。**

```css
@layer impA, impB;
@layer impA { .i { color: rgb(10,10,10) !important; } }
@layer impB { .i { color: rgb(20,20,20) !important; } }
/* 实测：rgb(10,10,10) —— 早声明的层在 !important 下反而赢 */
```

MDN 原文：*"Within author styles, all important declarations within CSS layers take precedence over any important declarations declared outside of a layer, while all normal declarations within CSS layers have a lower priority than declarations declared outside of a layer."* 本机两条实测都对上了：分层 `!important`（rgb(11,11,11)）压过未分层 `!important`（rgb(22,22,22)）。

**合起来有第四条**，也是最实用的一条：

**④ 分层里的 `!important` 能压过内联 `style`。**

```html
<style>@layer inl { .x { color: rgb(250,0,0) !important; } }</style>
<div class="x" style="color: rgb(0,200,0)">x</div>
<!-- 实测：rgb(250,0,0) —— 内联样式输了 -->
```

这条打破了"内联样式是不可战胜的"这个老印象：**内联样式只是普通优先级的"最高档"，遇上 `!important` 照样输**（`!important` 的层级本来就高于普通声明）。

再补两个语法点：**匿名层** `@layer { }` 按声明顺序排（实测后声明的那个赢）；**嵌套层**用点号 `@layer framework.layout { }`；`@import url("x.css") layer(vendor);` 可以把整个文件塞进某个层——这正是"给第三方库降级"的标准姿势。

### 9. `:has()`：CSS 终于有了"父选择器"和"前瞻"

`:has()` 的参数是**相对选择器**，所以它同时能往上和往右看：

```css
/* 选"包含 badge 的卡片"（往上选父元素） */
.card:has(> .badge) { padding: 0; }

/* 选"后面紧跟 .xyz 的 .abc"（前瞻，注意选中的是 .abc） */
.abc:has(+ .xyz) { margin-bottom: 0; }

/* 否定前瞻：.abc 后面不是 .xyz */
.abc:not(:has(+ .xyz)) { margin-bottom: 1rem; }
```

MDN 给了个好类比：`.abc:has(+ .xyz)` 等价于正则里的前瞻 `abc(?=xyz)`——**看后面，但选中前面那个**。

逻辑组合也直觉：`x:has(a, b)` 是 **OR**（有 a 或有 b），`x:has(a):has(b)` 是 **AND**（链式）。

### 10. `:has()` 的四条约束（每一条都有人答错）

**① 权重 = 参数里最具体的那个选择器**，跟 `:is()` / `:not()` 同一套规则。

本机实测：`.probe-has:has(div.a)`（= 0,2,1）**输给** `.probe-has.b.c`（= 0,3,0）。所以别指望用 `:has()` 提权，也不用担心它无脑提权。

对比一下 `:where()`：`.w:where(.a)` 的权重仍是 0,1,0（等于 `.w`），实测两个 `.w` 规则打成平手后由**源码顺序**决定——这就是 `:where()` 的用处（写"怎么都盖不过的重置样式"）。

**② 不能嵌套 `:has()`。** `:has(:has(div))` 不是合法选择器（本机 `CSS.supports('selector(:has(:has(div)))')` 实测为 `false`），"父选择器链"这种玩法不存在。

**③ 参数里不能用伪元素。** `div:has(::before)` 非法（实测 `false`）。MDN 解释了原因：很多伪元素是**根据祖先样式条件性存在**的，允许被查询会引入循环。

**④ 不支持时，整条规则直接失效。** 这是最容易踩工程坑的一条：CSS 选择器列表里只要有一个不支持的选择器，**整条规则**都会被丢弃。所以想兼容老浏览器，`no-has` 环境要把 `:has()` 塞进 `:is()` / `:where()` 这种**宽容列表**里（MDN 原文），或者干脆用 `@supports selector(:has(*))` 隔离开。

**性能**上 MDN 专门写了一节，结论可以直接背：

> In a selector like `A:has(B)`, make sure your `A` does not have too many children, and make sure your `B` is tightly constrained to avoid unnecessary traversal.

翻译成两条纪律：**锚定选择器别用 `body` / `:root` / `*` 这类"孩子太多"的元素**（任何 DOM 变更都要重查整棵子树）；**参数里用 `>`、`+` 收窄范围**（`article:has(> .title)` 比 `article:has(.title)` 省）。

## 其实你每天都在用

- **你写的每个 Web Component / Vue 组件库组件**，理想形态都是"内部用容器查询适配、外部什么布局都能塞"——这就是容器查询想解决的问题。
- **Tailwind v4 的产物文件开头**就是真实在用的 `@layer`：`@layer properties;` 和 `@layer theme, base, components, utilities;`，`utilities` 层能压过你手写的 `.card` 靠的就是它。
- **引入一份不带 `@layer` 的 reset.css**：它会压在你所有分层样式之上（第 8 节规则①），这就是很多人"引入 reset 后自己的样式莫名失效"的原因。
- **Element Plus 的 `el-button--primary`、Ant Design 的 `ant-btn-primary`**：BEM 那套命名思想在组件库里到处都是（下一篇正式讲）。
- **表单校验的视觉反馈**：`label:has(+ input:invalid)` 一行搞定"错误时标红"，以前要写 JS 监听。
- **表格/卡片的"和有图/无图两种形态"**：`.card:has(img)` 直接省掉一个 `has-image` 的修饰类，不用再让模板拼类名。
- **Chrome DevTools 的 Elements 面板**：容器元素旁边有个容器徽标，点亮它就能实时看 `@container` 什么时候命中——调容器查询基本都靠它。
- **设计稿里"卡片内容要和页面网格对齐"**：这就是子网格最早的商业场景，靠 `1fr` 硬凑宽度永远差几个 px。

## 常见误解（FAQ）

**❌ 误区一："有了容器查询，媒体查询就该淘汰了"**

两者问的是不同层级的问题。**页面骨架（列数、导航是展开还是抽屉、打印样式、`prefers-reduced-motion`）只能靠媒体查询**，因为那些决策依赖的是视口和设备特征，不存在"某个祖先容器"；**组件内部布局才该用容器查询**。成熟项目是两层都在用：外层 `@media` 定列宽，内层 `@container` 定卡片横排还是竖排。非尺寸类的媒体特性（`prefers-color-scheme`、`prefers-reduced-motion`、`print`）容器查询根本没有对应能力。

**❌ 误区二："`:has()` 的权重是 0，和 `:where()` 一样"**

反了，这是两个相反的极端。`:where()` 是**权重永远为 0**（实测 `.w:where(.a)` 输给同权重的 `.w`）；`:has()` 是**取参数里最具体的那个选择器作为自己的权重**（实测 `.probe-has:has(div.a)` = 0,2,1）。记混了会导致你对"谁能覆盖谁"的判断整个反过来。跟 `:where()` 同权的是 `:is()` / `:not()` 家族——它们权重都取"参数里最具体者"，只有 `:where()` 是 0。

**❌ 误区三："`@layer` 是用来控制打包/加载顺序的"**

层和文件、加载顺序**没有关系**。层的先后**只由第一次声明它的位置决定**（实测：重复声明只追加规则、不改顺序），跟这个层里的规则写在哪个文件、哪个 `<link>` 先加载无关。它是**层叠排序**机制，不是构建工具的配置项，也不会影响样式表的下载顺序。

**❌ 误区四："给容器加 `container-type` 是零成本的开关"**

它同时施加了 containment，**宽度靠内容撑的元素会塌成 0**（实测 inline-block / 绝对定位 / flex 子项都从 148.4px 变成 0px），`container-type: size` 连高度都会锁死（只有文字的 div 实测高度 0）。MDN 的原话是"拿不到上下文或显式尺寸时，带尺寸包含的元素会塌陷"。所以容器要挂在**块级元素、有明确尺寸的轨道、或者写了 `width` 的盒子**上，别挂在收缩包裹的元素上。

**❌ 误区五："`:has()` 不支持的浏览器加个 polyfill 就行"**

难。`:has()` 的语义是"依据后代状态决定祖先样式"，在纯 CSS 层没有等价降级手段，这类 polyfill 只能是 JS 遍历 DOM 打类名（成本高、还要监听所有变更）。更要命的是**级联安全**：如果写成 `p:has(.a), p.lead { ... }`，在不支持的环境中**整条规则**（包括本来没问题的 `p.lead` 部分）都会被丢弃。正确做法是 `@supports selector(:has(*))` 隔离，或者把 `:has()` 放进 `:is()` / `:where()` 这种宽容列表里。

**❌ 误区六："子网格就是更好用的嵌套 grid，随时能替"**

只在"**轨道数量确定**"时才好用。本机实测：子网格跨两行放 9 个子项，**多出来的 3 个直接挤在最后一条轨道上重叠**，容器高度完全不涨（90px）；同样的写法换成普通嵌套 grid，会新建隐式行、容器内容涨到 122px。所以子网格的代价是**放弃自动排布**——要放多少项、能不能整除，得先想清楚。另外它和 `display: contents` 也不是一回事：`contents` 是把盒子从布局里摘掉，子网格是**共享父级轨道但仍然是个 grid 容器**。

## 一句话总结

**容器查询让组件按"容器多大"适配（代价是包含会让容器塌陷）、子网格让嵌套 grid 复用父级轨道（代价是没有隐式轨道）、`:has()` 让选择器能往上和往前看（代价是整条规则不降级 + DOM 变更要重查子树）、`@layer` 让层叠冲突从"比权重"变成"比声明顺序"（代价是未分层样式永远最高、`!important` 还会反转层序）。**
