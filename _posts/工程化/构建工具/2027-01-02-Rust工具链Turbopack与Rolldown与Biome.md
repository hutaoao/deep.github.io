---
layout: post
title: "Rust 工具链：Turbopack / Rolldown / Biome"
date: 2027-01-02 00:00:00 +0800
categories: ["工程化", "构建工具"]
tags: [Rust, Rolldown, Turbopack, Biome, Oxc, 构建工具]
description: >
  前端工具链在 2024–2026 被整体重写了一遍：打包器、Linter、Formatter、类型检查器全换成了 Rust / Go。面试要答的不是"Rust 快"，而是这批工具换了什么架构、为什么换、以及代价是什么。
---

## 一句话概括

一句话说清这波变化：**前端工具链被重写了一遍，从「用 JS 写的工具」变成「用 Rust（少量 Go）写的工具」，而它们拿到的加速远远超过"换了个更快的语言"能解释的程度。**

事情的起点是 2020 年的 esbuild（Go 写的打包器），它第一次证明「打包器可以快一个数量级」。之后整个生态陆续跟进：打包器有 Rolldown、Rspack、Turbopack，Linter + Formatter 有 Biome 和 Oxc 的 oxlint/oxfmt，类型检查器有 TypeScript 的 Go 移植 tsgo。

面试问这个话题，一般不是想听你背工具名单，而是三层：

1. **现状层**——「Vite 8 现在底层是什么」「Next.js 16 为什么不用 webpack 了」（下面第 1~2 节）；
2. **原因层**——为什么 Rust 重写能快这么多（第 3 节，**这层最容易答浅，只答"Rust 比 JS 快"是不及格的**）；
3. **个体与取舍层**——Rolldown / Oxc / Turbopack / Biome 各自换了什么架构，这波重写打破了什么、迁移成本在哪（第 4~9 节）。

这篇按这三层来，中间会贴几个我在本机实测的数字（**测试环境的版本：Rolldown 1.2.13 / Vite 8.3.4 / esbuild 0.28.2 / Rollup 4.64.3 / Biome 2.5.15 / oxc-parser**）。

## 核心知识点

### 1. 先看这张 2026 工具链地图

| 岗位 | 旧工具（JS） | 新工具（Rust / Go） | 状态 |
|---|---|---|---|
| 应用打包 | Webpack、Rollup | **Turbopack**、**Rspack**、**Rolldown** | 已 GA |
| 库打包 | Rollup、tsup（基于 esbuild） | **Rolldown**、tsdown | 已 GA |
| 开发服务器 | Vite（内部用 esbuild） | Vite 8（内部 Rolldown + Oxc） | 默认 |
| Linter | ESLint | **Biome**、**oxlint** | 已 GA |
| Formatter | Prettier | **Biome**、**oxfmt** | 已 GA / beta |
| 类型检查 | tsc | **tsgo**（Go） | beta |
| 压缩 | Terser | **Oxc minify**、swc | 已 GA |
| 包管理 | npm、yarn | pnpm、Bun | 已 GA |
| Monorepo | Lerna、Nx（JS） | Turborepo、Nx（Rust core） | 已 GA |

**地图里只有两个名字属于 Go**：esbuild 和 tsgo。其余都是 Rust。答题时说「Rust 化」没错，但能被追问一句「esbuild 呢」时答出「Go，而且是这波的最早一批」，就显出你是真跟过。

### 2. 这几个主角其实不在同一层

这是最容易讲乱的地方，先把层分清：

| 工具 | 是什么 | 关键设计 |
|---|---|---|
| **Rolldown** | **打包器**，Vite 8 的统一内核 | Rust 实现 + **兼容 Rollup 插件 API** + 对标 esbuild 的功能面 |
| **Oxc** | **编译器底座**（不是产品，是工具集） | 一套共享 parser/AST 的 Rust 工具：parser、transformer、minifier、resolver、linter（oxlint）、formatter（oxfmt） |
| **Turbopack** | **打包器 + 增量计算引擎** | 基于 **turbo-tasks**（函数级 memoization），内置于 Next.js |
| **Biome** | **Linter + Formatter 一体**（一个二进制） | 单一 `biome.json`、单条 `biome check` 命令、架构参考 rust-analyzer |
| **Rspack** | 打包器，走**兼容 webpack** 路线 | 给大型遗留项目当低风险替代 |

一句话记它们的区别：

- **Rolldown 让你和 Rollup 的插件继续用**；
- **Oxc 让别人在上面再建工具**（Rolldown 的解析、转换、压缩、模块解析全是 Oxc）；
- **Turbopack 的卖点是"不重复计算"**，不是"跑得快"；
- **Biome 的卖点是"两个工具合成一个进程"**。

### 3. 为什么 Rust 重写能快：不要在"语言快"上停住

「Rust 比 JS 快，所以快」——这个答案会被追问到崩。真正的原因有三条，而且**第二条和第三条才是主要的**：

**① 能真并行。** JS 只有一条主线程能跑你的代码（worker 之间还要拷贝/序列化数据）；Rust 可以起真线程，多核直接摊平工作。前端工具链的活（解析、转换、压缩）几乎全是可以并行的数据并行任务。

**② 工具之间能共享同一个 parser 和 AST。** 这是最容易被忽略、但收益最大的一条。传统 JS 工具链里，Babel 解析一次、ESLint 解析一次、Terser 再解析一次，中间还不断把 AST 串成字符串再解析回来。Oxc 的定位就是「大家都用我这一个 parser」，于是同一份源码在整个流水线里只被解析一次。

**③ 能做真正的增量计算。** Turbopack 的 turbo-tasks 把编译拆成一堆小函数，每个函数的输出按内容寻址缓存，输入没变就不重算，还能把缓存持久化到磁盘（Next.js 16.1 起 dev 支持、16.3 起 build 默认打开）。这是 JS 工具链很难做好的事。

为了说明「单点提速 ≠ 端到端提速」，我实测了一下 Oxc 的 parser 对比：拿一个 **3.67MB 的真实文件**（`echarts.js`，未压缩源码）分别用三个 parser 解析：

```text
oxc-parser (Rust/napi)     33 ms / 次
@babel/parser 7.29.8      114 ms / 次   (3.5x)
acorn (JS)                106 ms / 次   (3.2x)
```

注意这个数字：**只有 3.5 倍——「Rust 化 = 快几十倍」这个印象，在单文件解析这一层并不成立。** 两个原因：

1. 这是**单文件、单线程**场景，没吃到多核；
2. oxc-parser 通过 napi 把结果序列化回 JS，**跨语言边界的成本**吃掉了一部分收益。

而 oxlint 端到端的差距能到几十倍，是因为它同时拿到了**多文件并行 + 只解析一次 + 结果不回传 JS** 这三份红利。**面试时如果你把单点 benchmark 当端到端结论讲，很容易被反问。** 反过来，你能主动说清这个区别，就是一个很强的加分点。

### 4. Rolldown：用 Rust 重写，但把 Rollup 的插件 API 留着

Rolldown 由 VoidZero（Evan You 成立的公司）开发，官方定位是「Rust-based bundler with **Rollup-compatible API** and esbuild feature parity」。

**它最重要的决定是「兼容 Rollup 插件 API」，而不是另起一套。** 因为 Rollup 的插件生态是它最大的资产——`vite.config.ts` 里那一串 `plugins`，绝大多数其实是 Rollup 插件。我实测了一下这个兼容性有多真：写一个标准的 Rollup 插件对象（`resolveId` + `load` + `transform` 三个钩子，用来提供虚拟模块和替换常量），**不改一行代码**分别喂给 Rollup 和 Rolldown：

```js
const plugin = {
  name: 'demo-logger',
  resolveId(source) { if (source === 'virtual:answer') return '\0virtual:answer'; },
  load(id) { if (id === '\0virtual:answer') return 'export const answer = 42;'; },
  transform(code, id) { if (id.includes('entry.js')) return code.replace('__VERSION__', '1.2.3'); },
};

await rollup({ input, plugins: [plugin] });     // ✅ 正常
await rolldown({ input, plugins: [plugin] });   // ✅ 同一个对象直接跑
```

两边产物措辞略有不同（Rolldown 会把内联结果做更彻底的常量折叠），但**插件契约是真兼容的**。

再看速度。我生成了一批互相引用的 ESM 模块（模拟中大型项目），分别打包：

| 模块数 | Rollup 4.64.3（JS） | esbuild 0.28.2（Go） | Rolldown 1.2.13（Rust） |
|---|---|---|---|
| 400 | 144 ms / 35.1 kB | 79 ms / 54.4 kB | **18 ms / 48.9 kB** |
| 2000 | 1620 ms / 186.7 kB | 86 ms / 286.1 kB | **80 ms / 254.2 kB** |

三个结论：

1. **Rolldown 和 esbuild 同一档**，比 Rollup 快约 **20 倍**（2000 模块：80ms vs 1620ms）——和 Rolldown 官网自报的 19k 模块 benchmark（Rolldown 1.61s / esbuild 1.70s / rspack 4.07s / Rollup+esbuild 40.10s）方向一致。时间线上：**Vite 8 于 2026-03-12 转正**，**Rolldown 1.0 于 2026-05-07 锁定稳定**。
2. **产物大小是另一回事**：Rollup 的产物最小（186.7kB），Rolldown 居中（254.2kB），esbuild 最大（286.1kB）。快和"产物最优"是两件事，别混着吹。
3. esbuild 的首次运行明显慢（79ms → 17ms），因为要起它的子进程/服务；**这点在真实项目里会被冷启动掩盖。**

**但 Rolldown 进 Vite 8 之后真正的收益不是速度，是 dev/prod 一致性。** Vite 7 及以前，dev 用 esbuild、build 用 Rollup——两套引擎、两个模块解析器、两套边缘 case，于是常年有一类"开发环境好好的、构建后崩了"的 bug。Vite 8 换成单引擎后，dev 和 prod 走同一条管线，**这一类 bug 从根上消失了**。官方给的理由也是这个：「两个打包器 = 两套插件系统 + 无穷的对齐胶水」。

本机 `vite@8.3.4` 的 `dependencies` 可以直接印证这件事：

```text
rolldown ~1.2.12 / lightningcss / postcss / picomatch / tinyglobby
```

**里面没有 rollup，也没有 esbuild。** 另外 `build.minify` 默认是 `'oxc'`（SSR 构建为 `false`），`build.rolldownOptions` 取代了 `rollupOptions`，`build.target` 默认是 `'baseline-widely-available'`——这几个都是可以直接答出来的具体变化。

### 5. Oxc：不显眼，但整条链都在用它

Oxc（Oxidation Compiler，名字是个双关：氧化就是生锈）是 VoidZero 的 Rust 工具集，组件齐全：

| 组件 | 对标 | 形态 |
|---|---|---|
| `oxc-parser` | `@babel/parser`、acorn | npm 包 + Rust crate |
| `oxc-transform` | Babel | 同上 |
| `oxc-minify` | Terser | 同上 |
| `oxc-resolver` | `enhanced-resolve` | 被 Nova / swc-node / knip 单独采用 |
| **oxlint** | ESLint | 独立 CLI，最成熟的组件 |
| **oxfmt** | Prettier | 独立 CLI，目前仍是 0.x / beta |
| 内部 transformer | — | Rolldown 的 JS/TS 转换与压缩都由它提供 |

**它的价值不在自己是个产品，而在「大家都用同一个 parser 和 AST」。** Vite 8 的 JS/TS 转换和 JS 压缩走的都是 Oxc，CSS 压缩交给 Lightning CSS——**只要你用 Vite 8，你已经在跑 Oxc 了，只是没人告诉你。**

这也是 Oxc 和 SWC / Biome 的定位差别：SWC 主要是转译器，Biome 是 linter + formatter，**Oxc 想做的是"所有上层工具共用的底座"**。

### 6. Turbopack：差异化是「函数级增量」，不是「Rust」

Turbopack 是 Vercel 的 Rust 打包器，历史轨迹很清楚：

| 版本 | 时间 | Turbopack 状态 |
|---|---|---|
| Next.js 13 | 2022-10 | `next dev --turbo` 实验 |
| Next.js 15.0 | 2024-10 | **`next dev` stable** |
| Next.js 15.5 | — | `next build --turbopack` beta |
| **Next.js 16.0** | 2025-10 | **dev + build 都稳定并成为默认** |
| Next.js 16.1 | 2025-12 | dev 的文件系统缓存稳定 |
| Next.js 16.3 | 2026-08 | **build 的磁盘缓存默认开启**，dev 内存下降最多 90% |

架构上三件事最重要（官方原文口径）：

1. **Unified Graph**——浏览器端、Node 服务端、Server Components 三种输出环境共用**一张模块图**，而不是跑多个编译器再拼产物；
2. **Incremental Computation**——「Turbopack parallelizes work across cores and caches results down to the **function level**. Once a piece of work is done, Turbopack won't repeat it.」这就是 turbo-tasks：把编译拆成小函数，按内容寻址缓存输出，只重跑输入变了的那些，缓存还能落盘跨进程复用；
3. **Lazy Bundling**——dev 只编译**被真正请求到**的路由。你一次会话里没打开的路由，压根不编译。

**HMR 快的原因就在第 2 条**：不是"Rust 反正快"，而是**真的少干了活**——只重算受影响的那一小片。

但它的代价也很硬，而且面试很爱追问：

| 限制 | 说明 |
|---|---|
| **没有 webpack 插件体系** | `next.config.js` 里的 `webpack()` 函数直接不生效；Next.js 16 起检测到它**会让 `next build` 报错**，而不是静默忽略 |
| 配置面收窄 | 用 `turbopack.rules` / `turbopack.resolveAlias` 替代原本的 loader 配置；生态里没有 Vite 那种插件 API |
| **构建期不能执行 JS 函数** | 典型例子：Sass 的 `sassOptions.functions` 官方明确不支持——「Turbopack's Rust-based architecture cannot directly execute JavaScript functions」 |
| 不支持 Babel 全量 | 用 SWC 做转换；只有检测到 Babel 配置文件时才启用 Babel |
| **不做类型检查** | `next build` 的 TS 检查是独立一步（16.3 起可以换用更快的 TS 7 编译器）；Turbopack 本身不查类型 |
| 部分平台无原生绑定 | FreeBSD / OpenBSD 只能退回 WASM，而 WASM 版**不支持 Turbopack**，只能 `--webpack` |
| 产物体积 | 目前仍略大于 webpack（差距在收敛），上生产前建议自己量一遍 |

「Turbopack 不能执行 JS 函数」这一条特别值得记——**它是所有 Rust 工具链的通病**：原生二进制里没法跑你的 JavaScript 插件。这直接引出了下一节的取舍。

### 7. Biome：一个二进制干掉 ESLint + Prettier

之前 ESLint 和 Prettier 是两套配置、两次运行，而且规则会互相打架（格式化改出来的样子不满足 lint 规则）。Biome 把两件事合成**一个二进制、一个 `biome.json`、一条 `biome check --write`**，架构参考 rust-analyzer，官方定位是「One toolchain for your web project」。

实测格式化 500 个 TypeScript 文件（每个都写得很脏：混乱缩进、缺失分号、多余空行）：

```text
biome format --write (Rust)     56 ms
prettier --write (JS)          831 ms
```

约 **15 倍**（官方自己的口径是在 2104 个文件 / 17.1 万行上比 Prettier 快约 35 倍，机器不同不必较真数量级）。另外顺手跑 `biome lint` 能同时报出 lint 问题。

版本上要记两个点：

- **Biome v2 于 2025-06-17 发布，代号 Biotype**，官方自称「the first JavaScript and TypeScript linter that provides **type-aware linting rules that doesn't rely on the TypeScript compiler**」——它自己做了一套类型推断，所以你可以不装 `typescript` 就做类型相关 lint。官方给的保守数字是：`noFloatingPromises` 这条规则能覆盖 typescript-eslint 约 **75%** 的场景（「your mileage may vary」），换来的开销小得多。
- v2 同时带来多文件分析、monorepo 支持、GritQL 插件系统、HTML formatter；官方首页目前列 **568 条规则**、与 Prettier **97% 兼容**（有明确的 known limitations 页面）。

```bash
# 典型落地姿势
npx @biomejs/biome migrate eslint --write   # 把 ESLint 配置迁过来
npx @biomejs/biome check --write ./src      # 格式 + lint 一次搞定
```

### 8. 代价清单：这波重写不是白拿的

这段是面试拉开差距的地方。Rust 化的收益是速度，成本是**生态断裂**，而且是真实存在的：

| 代价 | 具体表现 | 应对 |
|---|---|---|
| **JS 插件生态断裂** | 原生二进制跑不了你的 JS 插件：ESLint 里那些自定义规则、Babel 里的自定义 transform，在新工具里**没有等价物** | 纯语法规则绝大多数可以迁走；剩下那些需要类型信息的，写成独立小脚本（拿 TS Compiler API 单独跑） |
| **构建期不能调 JS** | 见上一节：Turbopack 不支持 `sassOptions.functions` | 保留 webpack 兜底，或改写为纯 CSS 方案 |
| **迁移会暴露边缘行为差异** | 依赖 Rollup / webpack 内部实现或时序 quirk 的插件最容易出问题 | 先跑 CI 全量验证，别在发版前夜升级 |
| **类型检查仍是独立一步** | 所有 Rust 打包器都**不做类型检查**（esbuild、Turbopack、Rolldown 都不做） | 并行跑 `tsc --noEmit`；要提速才上 tsgo |
| **平台绑定** | 原生二进制要匹配 OS + 架构；冷门平台只能退 WASM | 见 Turbopack 那条 |
| **生态碎片化** | 有的项目用 Oxc、有的用 SWC、有的用 Biome，规则和产物格式不统一 | 跟着主框架走（Vite 项目跟 Rolldown，Next.js 项目跟 Turbopack） |

### 9. 面试怎么答这个话题

按三层答，稳且不啰嗦：

**第一层，摆事实。** 「Vite 8 把 esbuild + Rollup 的双引擎换成了单一 Rust 打包器 Rolldown（Rust 实现、兼容 Rollup 插件 API），JS/TS 转换和压缩交给同门的 Oxc，CSS 交 Lightning CSS。Next.js 16 起 Turbopack 是 dev 和 build 的默认打包器，底层是 turbo-tasks 函数级增量计算。Lint 和 format 这边，Biome 和 Oxc 的 oxlint/oxfmt 在替代 ESLint + Prettier。类型检查这边 tsc 本身在往 Go 移植（tsgo）。」

**第二层，说原因（重点）。** 「速度不只是语言问题，主要是三条架构红利：能真并行、整条链**共享同一个 parser 和 AST**（不再每个工具各解析一遍 AST 还来回序列化）、以及**能做增量**（Turbopack 把编译切到函数级并做内容寻址缓存）。举个反例：我实测用 oxc-parser 解析 3.67MB 的单文件只比 Babel 快 3.5 倍——单点提速和端到端提速不是一回事。」

**第三层，讲取舍。** 「代价是插件生态断裂：原生二进制跑不了 JS 插件。所以迁移策略一般是先跑 oxlint 和 ESLint 并行、小项目直接上 Biome，构建侧跟着框架走；真正需要类型信息的自定义规则，就抽成独立脚本单独跑。另外要注意这波工具全都**不做类型检查**，`tsc --noEmit` 那一步跑不掉。」

## 其实你每天都在用

- `vite.config.ts` 里写的 `build.rollupOptions`，升级到 Vite 8 之后该叫 `build.rolldownOptions` 了。
- 升到 Next.js 16 第一次 `next build` 报错说不能用 `webpack()` 配置——Turbopack 不给静默忽略。
- 项目里装 `@biomejs/biome` 之后，`package.json` 里 `husky` 的 pre-commit 从跑两个命令变成一个 `biome check`，钩子时间肉眼可见地缩短。
- 你在 ESLint 里写过一个自定义规则，迁 Biome 时发现没等价物，只能降级成一个独立脚本——这就是生态断裂的具体样子。
- CI 的 lint 步骤从几十秒掉到一两秒（oxlint / Biome 大多数情况下）。
- 用 Next.js 开发时改一个组件的 HMR 感觉"瞬间生效"，因为只重算了那个模块的依赖闭包。
- 装 `@swc/core` 或者 `rolldown` 时报 `EBADPLATFORM` / 找不到原生绑定——那是在下平台专用的二进制。
- 单位里那位坚持"webpack 够用"的同事最近也开始问你 Vite 怎么配了。

## 常见误解（FAQ）

**❌ 误区一："Rust 写的就是快，因为 Rust 比 JS 快"**

这是最浅的答案。**速度主要来自架构红利：真并行、整链共享一个 parser/AST（省掉重复解析和跨工具序列化）、以及能做函数级增量。** 反证就是我上面那个实测：oxc-parser 单文件只快 3.5 倍。**如果加速只来自语言本身，端到端不该比单点高出那么多**——多出来的那一截，就是共享解析、并行和增量省下的重复劳动。

**❌ 误区二："Turbopack 和 Rolldown 是竞品，得二选一"**

不是同一层、也不是同一个使用场景。**Rolldown 是通用打包器**（Vite 8 的内核，也给库打包用），**Turbopack 是 Next.js 内置的打包器 + 增量计算引擎**（`turbo-tasks`）。你在 Vite 项目里用不到 Turbopack，在 Next.js 项目里也换不上 Rolldown。**真正同层的是 Rolldown / Rspack / Webpack / Rollup 这四个。**

**❌ 误区三："Biome 能完全替代 ESLint"**

核心规则和主流插件（TS、React、a11y、import 顺序那一批）覆盖得不错，官方首页列 568 条规则。**缺口在"需要类型信息 + 团队自定义"的那一小撮规则**：Biome 自己做了类型推断，但拿到的是它自己的类型信息，不是 TS 的类型系统；而原生二进制也没法跑你的 JS 插件。所以现实做法通常是**迁移核心规则 + 那几条特殊的抽成独立脚本**，而不是"一条命令全换完"。

**❌ 误区四："换成 Rust 打包器之后，类型检查也快了"**

**所有 Rust 打包器都不做类型检查**——esbuild 不做（官方明确「不做 TS 类型检查，要并行跑 `tsc -noEmit`」）、Turbopack 不做、Rolldown 也不做。它们只是**转译**（剥掉类型）。想提速类型检查是另一件事：那是 **tsgo**（TypeScript 编译器的 Go 移植）的目标，大项目上快约 10 倍，但直到 2026 年仍是 beta。

**❌ 误区五："Vite 8 只是把 Rollup 换成 Rolldown，行为不变"**

**引擎一换，边缘行为就会变**，特别是那些依赖 Rollup 内部实现、文档没写的时序 quirk 的插件。Vite 8 会尽量自动把 `esbuild` 和 `rollupOptions` 配置转成 Rolldown / Oxc 的等价物，但"大多数项目一次构建通过"和"行为完全一致"是两件事。**升级前跑一遍完整 CI，别拿生产环境试。**

**❌ 误区六："Rolldown 是 Rollup 的 Rust 版，所以就是同一个东西换个语言"**

措辞要准：Rolldown 是 **Rust 从零实现的打包器，同时兼容 Rollup 的插件 API**，不是把 Rollup 的代码翻译成 Rust，也不是同样的维护团队（Rolldown 属于 VoidZero）。这个区别很实质——它决定了插件能直接用，但**内部行为不保证一致**。

**❌ 误区七："oxlint 比 ESLint 快几十倍，所以解析单文件也一样快几十倍"**

**端到端倍数 ≠ 单点倍数。** oxlint 端到端的收益来自多文件并行 + 不重复解析 + 结果不跨语言回传；单文件解析器层面，实测只有 3～4 倍（还含 napi 序列化开销）。答面试题时把这两个数字混着用，很容易被反问。

**❌ 误区八："SWC 也是 Rust 写的，跟这几个是一批竞品"**

SWC **主要是转译器**（Babel 的替代品），它跑在 Webpack / Rspack / Jest / Next.js 里当"语法转换那一环"，而不是自己管整条打包流水线。它和 Oxc 的 transformer 部分同层，和 Rolldown / Turbopack 不在同一层。

## 一句话总结

**这波重写换的不只是语言，是架构：Rolldown 用 Rust + Rollup 插件 API 把 Vite 的 dev/prod 两条管线并成一条，Oxc 让整条链共享一个 parser，Turbopack 靠函数级增量"少干活"，Biome 把 lint 和 format 压进一个二进制——快来自并行、共享解析与增量这三条，而不是"Rust 天生快"；代价是原生二进制跑不了 JS 插件，所以它是一次渐进的生态迁移，不是一次替换。**
