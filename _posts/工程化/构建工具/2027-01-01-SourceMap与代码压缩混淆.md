---
layout: post
title: "Source Map 与代码压缩混淆"
date: 2027-01-01 00:00:00 +0800
categories: ["工程化", "构建工具"]
tags: [源码映射, 代码压缩, Terser, 代码混淆, 构建工具]
description: >
  Source Map 是一份「生成代码位置 → 源码位置」的字典，用 Base64 VLQ 加相对偏移压成了一行字符串。面试问它通常是三层：格式怎么读、地图怎么串起来、生产环境怎么部署——而最容易答错的一条是「压缩不等于混淆」。
---

## 一句话概括

Source Map 就是一份**字典**：给出「打包后文件第 X 行第 Y 列」，它能告诉你「这是源码文件 A 的第 M 行第 N 列」。

说穿了它要解决一个问题：**线上代码已经被压成了一行、变量名也改成了 `t`、`e`、`n`，报错栈完全看不懂。** Source Map 就是在构建时顺手记下的对照表，让 DevTools 和错误监控平台能把 `app.min.js:1:38421` 翻译回 `src/lib/pricing.ts:43:14`。

它一共只有 8 个字段，其中 7 个一看就懂，真正需要讲清楚的是 `mappings`——用 Base64 VLQ 编码 + 相对偏移压成了一行字符串，为了让 map 不至于比代码本身还大。

面试问这个知识点通常是三层：

1. **格式层**——`mappings` 那一串 `AAAA,SAASA` 到底怎么读（下面第 1~2 节）；
2. **语义层**——压缩到底改了什么名字、压缩为什么不是混淆（第 3~4 节）；
3. **链路与工程层**——TypeScript → JavaScript → minify 地图怎么串、`devtool` 那串 `hidden-cheap-module-source-map` 是什么意思、生产该选哪个（第 5~7 节）。

但真正拉开差距的，是旁边那个常被一起问的话题：**压缩（minify）和混淆（obfuscate）不是一回事**，而且压缩从来不是安全手段。这条答错，基本就说明只配过配置、没想过原理。

## 核心知识点

### 1. 一份 Source Map 只有 8 个字段，核心只有一个

先看真实产物。用 Terser `5.51.2` 压一段代码并生成 map，得到的东西是这样：

```json
{
  "version": 3,
  "file": "price.min.js",
  "sources": ["src/price.js"],
  "sourcesContent": ["function calculateDiscount(userTier, cartTotal) { ... }"],
  "names": ["calculateDiscount", "userTier", "cartTotal", "total", "console", "log"],
  "mappings": "AAAA,SAASA,kBAAkBC,EAAUC,GAEnC,OAAOA,GADoB,QAAbD,EAAqB,GAAM,EAE3C,..."
}
```

| 字段 | 必需 | 作用 | 面试要点 |
|---|---|---|---|
| `version` | ✅ | 格式版本 | **恒为整数 3**。ECMA-426 明确「不再有版本号」，写别的值消费方可以拒绝 |
| `file` | ❌ | 生成文件名 | 只是给工具看的，不参与路径解析 |
| `sourceRoot` | ❌ | sources 的公共前缀 | 会和 `sources` 拼起来 |
| `sources` | ✅ | 原始文件路径数组 | **相对 map 文件自身路径解析**，不是相对页面 |
| `sourcesContent` | ❌ | 源文件内容 | 「源码直接躺在 map 里」——泄露的根源 |
| `names` | ❌ | 原始标识符数组 | 被 `mappings` 第 5 个值索引，压缩改名后靠它还原名字 |
| `mappings` | ✅ | 核心，Base64 VLQ 串 | 唯一的难点 |
| `ignoreList` | ❌ | 第三方源码下标 | 让 DevTools 默认折叠 `node_modules`（旧名字 `x_google_ignoreList` 已废弃） |

**面试最容易漏掉的是 `sourcesContent`。** 它把原始源码**原文**塞进了 map。所以「我把代码压缩了，别人看不出业务逻辑」这个想法，在你部署 `.map` 的那一秒就破产了——`sourcesContent[0]` 就是完整的 TypeScript 源文件。

### 2. `mappings` 怎么读：分号分行、逗号分段、段内 1/4/5 个数

规范（ECMA-426 §9.2）对结构的描述其实很短，就三句话：

- **分号 `;`** 分隔生成代码的每一行；
- **逗号 `,`** 分隔同一行内的各个 segment；
- 每个 segment 由 **1、4 或 5 个变长字段**组成。

字段按顺序是这五个：

| 序号 | 含义 | 相对谁 |
|---|---|---|
| 1 | 生成代码中的列号 | 相对**本行上一个 segment**；换行后重置 |
| 2 | `sources` 数组下标 | 相对上一次出现该字段的值 |
| 3 | 源码行号 | 同上 |
| 4 | 源码列号 | 同上 |
| 5 | `names` 数组下标（可选） | 同上 |

三个细节决定了它能不能被压缩得这么小：

1. **除了第一个字段，其余都是 delta（相对上一次的值）**，不是绝对值。这样绝大多数数字都落在 `0~15`，一个字符就够。
2. **第 1 个字段换行会重置，其余不会。** 规范的原文专门点了这一条：列号在每行 `;` 之后重新计数，所以它相对的是「本行上一个 segment」；而源文件下标、源行、源列是**跨行累积**的相对值。
3. **1 个字段的 segment 代表「没有对应源码」**，是编译器自动生成的代码；4 个字段代表有映射但没名字；5 个字段代表连名字都有。

拿上面那份真实 map 解码看（`@jridgewell/sourcemap-codec`）：

```text
原始: "AAAA,SAASA,kBAAkBC,EAAUC,GAEnC,..."
解码: [[0,0,0,0], [9,0,0,9,0], [27,0,0,27,1], [29,0,0,37,2], [32,0,2,2], ...]
```

怎么读第二个 segment `[9,0,0,9,0]`？

- 生成代码第 **9** 列；
- 源文件下标 **0**（`src/price.js`）；
- 源码第 **0** 行；
- 源码第 **9** 列；
- 名字下标 **0** → `names[0]` = `calculateDiscount`。

也就是说：生成文件的 `第 9 列` 对应源码里 `calculateDiscount` 这个标识符。**哪怕 Terser 把它改名成 `n`（上面 `toplevel` 那份产物就是），`names` 表里照样记着原名**——这就是报错栈能显示原名、DevTools 能下断点的原理。

顺带一个能证明「VLQ 到底省了多少」的历史数字：v3 的 VLQ + 段式编码，相比 v2 提案大约又省了 50%（规范 Note 1 里给的实测样本是 Google Calendar）。

### 3. 压缩时名字到底被改了什么：默认只改局部

很多人以为「压缩 = 所有名字都变短」，实测（Terser `5.51.2`）说明不是：

```js
// 源码
function calculateDiscount(userTier, cartTotal) {
  const ratio = userTier === 'vip' ? 0.8 : 1;
  return cartTotal * ratio;
}
const total = calculateDiscount('vip', 100);
console.log(total);
```

```js
// 默认 minify()：只改函数参数
function calculateDiscount(t,c){return c*("vip"===t?.8:1)}const total=calculateDiscount("vip",100);console.log(total);

// mangle: { toplevel: true }：连顶层一起改
function n(n,o){return o*("vip"===n?.8:1)}const o=n("vip",100);console.log(o);
```

**关键结论：`mangle` 默认不动顶层作用域的名字。** 顶层变量、导出名、全局函数名必须加 `toplevel: true` 才改名——因为库的导出名一改，外部就没法按名调用了。

还要分清这几件事（都容易混）：

| 选项 | 干什么 | 效果 |
|---|---|---|
| `compress` | 改**代码结构**：常量折叠、内联、删死代码 | `const ratio = ...; return cartTotal * ratio` 被内联成一句 |
| `mangle` | 改**标识符名字** | `userTier` → `t` |
| `mangle.toplevel` | 允许 mangle 顶层名 | `calculateDiscount` → `n` |
| `keep_fnames` | 保住函数名 | 上方示例里 `calculateDiscount` 未被改 |
| `mangle.properties` | 改**属性名**（默认关闭） | `obj._meta` → `obj.t` |

注意 `mangle: false` 的产物里 `ratio` 依然消失了——那是 `compress` 干的，不是 mangle。**「压缩」是两件事的组合：结构化简 + 改名。**

### 4. 压缩 ≠ 混淆：这条最容易答错

面试官问「代码压缩能起到混淆作用吗」，标准答案是：**方向对，但完全不够，压缩从来不是安全手段。**

实测同一个文件的产物：

```js
// 输入（含一个密钥字符串）
const config = { _userId: 42, _userName: 'admin', apiKey: 'sk-live-abcdef123456' };

// 压缩后（节选）
const o=42;export const token="eyJhbGciOiJIUzI1NiJ9.payload";
```

- **字符串常量原样保留**：JWT、接口路径 `/api/user/`、错误文案，一个都没被加密或编码；
- **业务逻辑结构可读**：`fetch("/api/user/"+o)` 一眼能看出在干什么（`config` 对象被内联掉、`apiKey` 被当死代码删了——那是 `compress` 在优化，不是在隐藏）；
- 真混淆（字符串数组化 + 控制流平坦化 + 死代码注入）才做这些事，Terser/esbuild 都不干。

属性名混淆还有额外的坑。实测：

```text
// 开启 mangle.properties: { regex: /^_/ } 后
user._meta.name        →  r.t.name        // ✅ 点号访问同步改名
user['_meta']          →  r.t             // ✅ 字面量字符串键也会改
user[key]              →  r[e]            // ❌ 动态访问改不了，直接失配
```

**最后一行就是属性混淆最经典的翻车点**：只要有一处是通过变量动态取属性名，改名后那一处就永远读不到了。所以默认 `mangle.properties` 是关的；开了就必须配 `regex` 限定范围，配 `reserved` 保护不能动的名字，并且保证没有动态访问。

最后，**只要 map 还在，改名就是可逆的**。map 里带着 `names` 表和 `sourcesContent`，任何拿到 map 的人都能一键还原源码。想靠压缩保密，等于把门锁上然后把钥匙贴在门上。

### 5. 多级映射：TS → JS → minify，地图怎么串起来

真实构建至少两级转换：`tsc`/esbuild 先把 TS 变 JS（生成 map A），Terser 再压缩（生成 map B）。**map B 的 `sources` 该指向 `.js` 还是 `.ts`？** ECMA-426 Annex B.2 把这件事说得很直白：

> It is getting more common to have tools generate sources from some DSL (templates) or compile TypeScript → JavaScript → minified JavaScript, resulting in multiple translations before the final source map is created. This problem can be handled in one of two ways. The easy but lossy way is to ignore the intermediate steps...

规范给了两条路：**省事的做法**是忽略中间产物（把中间文件当"源文件"）；**完整的做法**是把上一级的 map 传下去，链式还原。

实测两级串联（esbuild `transform` 出第一级 map，喂给 Terser）：

```js
// 第一级：esbuild 把 calc.ts 转成 JS，产出 map（sources: ["calc.ts"]）
const s1 = await transform(tsCode, { loader: 'ts', sourcemap: true, sourcefile: 'calc.ts' });

// 第二级：把上一级 map 作为 content 传进去
const s2 = await minify({ 'calc.js': s1.code }, {
  sourceMap: { content: s1.map, filename: 'calc.min.js', includeSources: true },
});
```

```text
传 content  →  sources: ["calc.ts"]，sourcesContent 是 TS 原文，断点直接落在 .ts 上 ✅
不传 content →  sources: ["calc.js"]，只能定位到中间产物 ❌
```

**这就是「生产环境断点落在 .js 而不是 .ts / 报错栈指向 webpack 中间文件」的根本原因**：不是浏览器坏了，是构建链路上某一环没把上一级的 map 接上。常见的三个断点：

1. Loader / 转译器没开 `sourceMap: true`，第一级根本没产物；
2. 压缩器没收到 `sourceMap.content`（Webpack 需要 `minimizer` 配 `sourceMap: true`，官方文档专门有一句提醒）；
3. 中间插了一步字符串拼接 / 手写模板替换，那一步没有 map，链路直接断在这里。

### 6. `devtool` 那串名字怎么读

Webpack 的 `devtool` 看着像乱码，其实是一个**固定顺序的拼接模板**（官方原文）：

```text
[inline-|hidden-|eval-][nosources-][cheap-[module-]]source-map[-debugids]
```

| 关键字 | 位置 | 含义 |
|---|---|---|
| `inline-` | 最前 | map 以 Data URL 内联进产物，不单独出文件 |
| `hidden-` | 最前 | **生成 map 但不写 `//# sourceMappingURL` 注释** |
| `eval-` | 最前 | 每个模块用 `eval()` 包起来，map 挂在 eval 上（开发快） |
| `nosources-` | 第二段 | 不带 `sourcesContent`，只留文件名和结构 |
| `cheap-` | 第三段 | **只有行映射，没有列映射**，断点无法落在行中间 |
| `module-` | 紧跟 cheap | 把 loader 的 map 也一起吃进来（否则只映射转译后的代码） |
| `source-map` | 必须 | 主类型 |
| `-debugids` | 最后 | 给产物打 Debug ID 用（配合 Sentry 一类错误监控平台的 map 匹配） |

**顺序不能错**，官方明说会校验这个模式。

质量分五档，名字其实就是它给你的东西：

| 档位 | 你看到什么 | 谁给的 |
|---|---|---|
| bundled code | 一坨打包产物 | 无 devtool |
| generated code | 能分模块，但是 Webpack 转译后的代码 | `eval` |
| transformed code | loader 转译后的代码，**只有行** | `cheap` |
| original lines | 你写的源码，**只有行** | `cheap-module` |
| original | 你写的源码，行列都有 | `module` / 默认 |

生产环境实际只有三个候选，官方每一条都配了明确的话术：

| devtool | 带注释 | 带源码 | 官方定位（原文要点） |
|---|---|---|---|
| `source-map` | ✅ | ✅ | 高质量生产 map 的推荐选择；**但你要配服务器禁止普通用户访问 .map** |
| `hidden-source-map` | ❌ | ✅ | 「只用 SourceMap 做错误上报」时用。原文警告：**不要把 map 部署到 webserver** |
| `nosources-source-map` | ✅ | ❌ | 可以部署到服务器。原文：**它仍然暴露文件名和目录结构，但不暴露原始代码** |

一句话记：**`hidden` 管"别引用"，`nosources` 管"别放源码"，两者解决的是不同问题，可以叠着用。**

### 7. 分环境怎么选

| 环境 | 选择 | 理由 |
|---|---|---|
| 本地开发 | `eval-cheap-module-source-map` | 重建最快，还能映射到源码行 |
| 本地调试复杂问题 | `eval-source-map` | 换来完整行列映射，代价是首次构建慢 |
| 预发 / QA | `source-map`（内网） | 要真实调试体验，又不能外网可见 |
| 生产 + 错误监控 | `hidden-source-map`，map 上传到监控平台后从产物里删掉 | 线上排错能力保留，源码不落地 |
| 生产 + 必须自带 map | `nosources-source-map` | 只暴露文件名和结构 |
| 生产 + 不要 | `false`（`devtool` 的默认值；`mode: 'development'` 下才会默认成 `'eval'`） | 官方称这是「可以起步的选择」 |

另外两条 2026 年值得知道的事实：

- **Source Map 已经不是约定俗成的野格式了**。ECMA-426 第 1 版于 **2024 年 12 月**发布，此后由 TC39-TG4 维护。规范里连 `//# sourceMappingURL` 的提取正则都写好了：`^[@#]\s*sourceMappingURL=(\S*?)\s*$`，并明确 **生成方只该写 `//#`，消费方必须同时接受 `//@`**（`//@` 是历史原因，和 IE 的条件编译冲突才换的）。顺带还有 `sourcemap` HTTP 响应头这条获取通道（旧的 `x-sourcemap` 已废弃）。
- 新特性在推进中：**Scopes 提案 Stage 3、Range Mappings / Debug ID Stage 2、Env Stage 1**。Debug ID 就是为了解决「产物 hash 变了，监控平台对不上 map」这个运维痛点——而 Webpack 的 `-debugids` 后缀就是它的落地形态。

## 其实你每天都在用

- 打开 DevTools 看 `Sources` 面板时多出来的 `webpack://` 目录树，就是 map 里的 `sources` + `sourceRoot` 拼出来的。
- 在压缩后的一行代码里下断点，断点却停在你写的 `.ts` 文件第 42 行——`mappings` 在工作。
- 线上报错 `Uncaught TypeError: Cannot read properties of undefined (reading 'calculateDiscount')`，监控平台能告诉你它在 `pricing.ts` 第 43 行——后端拿到了 `.map` 做的还原。
- 改一行代码，dev server 立刻热更新且断点位置不乱——HMR 会顺带把新模块的 map 一起推过来。
- `vite.config.ts` 里写的 `build: { sourcemap: 'hidden' }`，就是 Vite 版的 `hidden-source-map`（`true` / `false` / `'inline'` / `'hidden'` 四个值）。
- 线上调试面板里 `node_modules` 被折叠成灰字，鼠标点不开——`ignoreList` 生效了。
- 打包后点开 `.map` 吓了一跳：400KB 的产物配了 1.5MB 的 map——因为 `sourcesContent` 把整份源码都抄进去了。
- 埋点 SDK 报错信息里只有 `at t (app.min.js:1:1204)`，你先去翻 map 才发现对不上号——那是上次发版忘了同步上传新 map。

## 常见误解（FAQ）

**❌ 误区一："`//@ sourceMappingURL` 是过时写法，应该全改成 `//#`"**

半对。规范明确：**生成方只该输出 `//#`，但消费方要同时接受 `//@`**。所以你自己生成时写 `//#` 没问题，但如果你在写一个解析器/监控 SDK，只认 `//#` 就会漏掉老工具产出的 map。

**❌ 误区二："`.map` 和产物一起传 CDN 就行，DevTools 体验更好"**

这是最危险的一条。map 里带 `sourcesContent`，等于把源码一并发布。Webpack 官方对 `source-map` 的提示原文就是「You should configure your server to disallow access to the Source Map file for normal users」，对 `hidden-source-map` 更是直接说不要把 map 部署到服务器。**正确流程是：CI 构建时生成 map → 上传到错误监控平台 → 从部署产物里删掉。**

**❌ 误区三："`mappings` 里存的是绝对行列号"**

不是。**除第一个字段外全是相对上一个值的偏移（delta）**，而且生成列号在换行后重置、其余字段跨行累积。这个设计的唯一目的就是压缩体积：正是因为它，一份 200KB 产物的 map 通常是 1~2MB 而不是十几 MB。

**❌ 误区四："压缩把变量名改短了，业务逻辑就安全了"**

改的只有标识符，**字符串常量、请求路径、业务分支结构一个都不动**。真要防逆向得上字符串数组化、控制流平坦化那套真混淆，代价是体积和运行速度。而且**只要 `.map` 泄露，`names` + `sourcesContent` 能一键还原全部源码**。

**❌ 误区五："开了 `mangle` 就所有名字都变短了"**

默认只改**函数作用域内**的名字。顶层变量、导出名、全局函数名需要 `toplevel: true` 才动；函数名可以用 `keep_fnames` 保回来。这也是为什么压缩后 `calculateDiscount` 有时还在、有时变成 `n`——取决于 `toplevel`。

**❌ 误区六："`cheap` 是质量差，生产环境别用"**

`cheap` 的准确含义是 **只有行映射、没有列映射**，它换的是「重建速度」。而它真正的硬限制是另一条：**`cheap` 不能和压缩共存**，因为压缩器通常把整个文件输出成一行，没有列信息就等于全文件只有一条映射。所以它天生只属于开发环境。

**❌ 误区七："`hidden-source-map` 就安全了"**

`hidden-` 只做一件事：**不在产物里写 `//# sourceMappingURL` 注释**。它照样生成带 `sourcesContent` 的 `.map` 文件。如果你把它也部署上去了，任何人把文件名拼一下（`app.js` → `app.js.map`）就能拿到——安全性完全来自"你没部署"，不来自这个选项。

**❌ 误区八："Source Map 就是个社区约定"**

**它已经是标准了**：ECMA-426 第 1 版 2024 年 12 月发布，TC39-TG4 维护，规范连 `sourceMappingURL` 的提取正则、`version` 恒为 3、消费方必须忽略不认识的字段都写进去了。答题时主动提一句，是个不小的加分点。

## 一句话总结

**Source Map 就是一份用 Base64 VLQ 和相对偏移压扁的「生成位置 → 源码位置」字典（ECMA-426，`version` 恒为 3）；它的价值全在链路完整（多级转换要逐级传 map，否则断链），它的风险全在 `sourcesContent`（源码原文）；而压缩只是改名和化简，`mangle` 不是加密，`.map` 一泄露，混淆白做。**
