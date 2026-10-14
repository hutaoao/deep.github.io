---
layout: post
title: "幽灵依赖与 peerDependencies：依赖提升的失效模式"
date: 2027-01-03 00:00:00 +0800
categories: ["工程化", "构建工具"]
tags: [pnpm, npm, 幽灵依赖, peerDependencies, 依赖管理]
description: >
  npm 的 hoisting 为什么会造出幽灵依赖、pnpm 的隔离为什么只是"半严格"、peerDependencies 为什么会造出同一个包的两个实例——
  全部用本机 npm 10 与 pnpm 11 实装对照给出证据，并附一份"依赖解析出问题"的排查清单。
---

## 一句话概括

**幽灵依赖（phantom dependency）** 指的是：你的代码 import 了一个 `package.json` 里根本没声明的包，居然能跑。这不是玄学，是 npm 的扁平化安装（hoisting）"顺手"把间接依赖也放到了顶层 `node_modules` 里。

面试问这个，真正想听的是三件事：

1. **提升规则本身**——为什么 npm 不保证"一个包在树里只有一份"；
2. **pnpm 到底隔离住了什么**——很多人的答案是"pnpm 彻底消灭了幽灵依赖"，这句话只对一半（第 3 节）；
3. **peerDependencies 为什么是多实例的源头**——React 双实例报 `Invalid hook call` 就是它干的。

下面所有结论都来自本机实测（npm 10.9.7 / pnpm 11.5.2），不是背来的。

## 核心知识点

### 1. 幽灵依赖的根因：hoisting 是"能提就提"

npm 从 v3 起把嵌套结构改成扁平结构：安装时**尽量把依赖往顶层 `node_modules` 放，只有遇到版本冲突才在子目录里嵌套一份**。代价是顶层从此挤满了你根本没装过的包。

实测对照（两边都是 `npm install express@4.18.2` / `pnpm install express@4.18.2`）：

| 指标 | npm | pnpm |
|---|---|---|
| 顶层 `node_modules` 条目数 | **67** | **1**（只有 `express` 一个符号链接） |
| `node_modules/accepts` 存在吗 | 存在 | 不存在 |
| 项目根 `require.resolve('accepts')` | 成功 | `MODULE_NOT_FOUND` |

于是同一行代码，两种命运：

```js
// package.json 里只声明了 express，没声明 accepts
const accepts = require('accepts')

// ❌ npm 下能跑——因为 accepts 被提升到了顶层
//    隐患：express 哪天不再依赖 accepts，这行就炸了，而且是在生产环境炸
// ✅ pnpm 下直接 MODULE_NOT_FOUND，在本地就把问题暴露出来
```

**面试可以直接说的结论**：幽灵依赖不是 bug，是扁平化的副产品；它真正的危险在于**失败延后**——问题不在你写这行代码时暴露，而在某次依赖升级后暴露。

### 2. 版本冲突时提升会"嵌套"：谁被嵌进去，谁就解析到那个版本

顶层一个包名只能放一个版本，剩下的版本会被嵌到需要它的那个包里。实测（同时装 `debug@2.6.9` 和 `ms@2.1.3`，而 `debug@2.6.9` 自己依赖 `ms@2.0.0`）：

```text
npm 的产物
node_modules/
├── debug/                     ← 顶层
│   └── node_modules/ms/       ← ms@2.0.0，被嵌套在这里
└── ms/                        ← ms@2.1.3（直接依赖占顶层）

实测：debug 内部 require.resolve('ms')
  → node_modules/debug/node_modules/ms/index.js   （拿到 2.0.0，正确）
```

pnpm 装同样的两个包，结构完全不同：

```text
pnpm 的产物
node_modules/
├── debug -> .pnpm/debug@2.6.9/node_modules/debug
├── ms    -> .pnpm/ms@2.1.3/node_modules/ms
└── .pnpm/
    ├── ms@2.0.0/               ← 两个版本都在，互不覆盖
    ├── ms@2.1.3/
    └── debug@2.6.9/node_modules/
        └── ms -> ../../ms@2.0.0/node_modules/ms   ← 用符号链接指过去

实测：debug 内部 require.resolve('ms')
  → node_modules/.pnpm/ms@2.0.0/node_modules/ms/index.js （也是 2.0.0）
```

两边**解析结果一样**，差别在形态：npm 靠"物理嵌套"（`debug` 里真的塞了一份 `ms`），pnpm 靠"符号链接重定向"（冲突版本放在不同的 `.pnpm/<包>@<版本>/` 目录里，再用符号链接指过去）。所以 pnpm 里每个包的**物理文件只有一份**，不会出现同一个包在多处各躺一份的情况。

顺带纠正一个流传很广的说法：**pnpm 并不让路径更短**。实测解析同一个 `debug`，npm 路径 58 字符、pnpm 90 字符（`.pnpm/<包>@<版本>/node_modules/<包>` 这层固定前缀更长）。pnpm 在 Windows 上的实际收益是**不重复落盘**，它自己反而需要 `virtualStoreDirMaxLength` 这类设置来规避路径过长。

### 3. pnpm 的隔离边界：官方口径是"半严格（semistrict）"

这是本文最值得记住的一点。pnpm 的隔离只对**项目代码**成立，对**依赖自己的代码**不成立。

pnpm 官方文档把默认布局叫 **semistrict**，原文：*"dependencies have access to undeclared dependencies but modules outside of `node_modules` do not"*。原因就是那个隐藏的提升层 **`node_modules/.pnpm/node_modules`**（`hoistPattern` 默认 `['*']`）。

实测三问（同一个 pnpm 项目，装了 express）：

| 从哪儿访问 | 目标 | 结果 |
|---|---|---|
| 项目根（你的业务代码） | `express` | ✅ 可访问 |
| 项目根（你的业务代码） | `accepts` | ❌ `MODULE_NOT_FOUND` |
| `node_modules/.pnpm/accepts@1.3.8/...`（依赖内部） | `express`（自己的 package.json 里没声明） | ⚠️ **能 resolve 到** |

那个隐藏提升层里躺了 **66 个条目**，数字对得上：`.pnpm/` 下共 70 个带版本的包目录，其中 `mime-db`、`mime-types`、`ms` 各有 2 个版本，**同名在提升层只能留一个**（70 − 3 = 67），再减去直接依赖 `express` 自己（不提升），正好 66。

```text
node_modules/
└── .pnpm/
    ├── node_modules/      ← 隐藏的提升层：间接依赖都在这里（默认行为，hoistPattern 默认 ['*']）
    │   ├── accepts -> ../accepts@1.3.8/node_modules/accepts
    │   ├── mime-db -> ../mime-db@1.40.0/node_modules/mime-db      ← 同名多版本，只留一个
    │   └── ms      -> ../ms@2.0.0/node_modules/ms                 ← 且留哪个没有语义保证
    └── express@4.18.2/node_modules/express/
```

注意提升层里用的是**不带版本号**的目录名（`accepts` 而不是 `accepts@1.3.8`），所以同名多版本必然要折叠。实测留下来的版本高低不一：`mime-db` 留的是 1.40.0、`mime-types` 留的是 2.1.34、`ms` 留的是 2.0.0——**留哪个由 pnpm 内部顺序决定，不承诺取新或取旧**。

**面试怎么答**：pnpm 消除的是"**业务代码**的幽灵依赖"；依赖自己出错时仍能通过隐藏层解析到未声明的包，而且**解析到哪个版本同样不确定**——本质上是把 npm 的不确定性从"顶层可见"挪到了"隐藏层里"。真要拿到根目录（很多老工具只认根 `node_modules`），才需要 `publicHoistPattern` 或 `shamefully-hoist=true`——注意后者的官方描述就是"把幽灵依赖请回根目录"，只能当临时手段。

### 4. peerDependencies：多实例的源头

`peerDependencies` 的语义是「我是插件，宿主必须提供这个包」——**声明的是兼容范围，不是让包管理器自己去装一份**。

npm 官方文档写得很明确：*"In npm versions 3 through 6, `peerDependencies` were not automatically installed... As of npm v7, `peerDependencies` **are** installed by default."* pnpm 侧对应的是 `autoInstallPeers`，官方默认值 `true`（取满足范围的最高版本）。所以"peer 只是提醒、不会真装"这个答案在 npm 7+ 已经过期了。

实测「只声明 `react-dom@19.2.0`、完全不声明 `react`」：

```text
npm 顶层：
  node_modules/react      ← 自动装进来了，版本 19.3.0（满足 ^19.2.0 的最高版）
  node_modules/react-dom

pnpm 顶层：
  node_modules/react-dom  ← 只有它一个符号链接，没有 react
  node_modules/.pnpm/react-dom@19.2.0_react@19.3.0/   ← react 在这层里
```

请注意 pnpm 那个目录名 **`react-dom@19.2.0_react@19.3.0`**——`_` 后面跟的是这个包解析到的 peer。**同一个版本的包，被解析到不同的 peer，就是不同实例**，官方文档管这个叫 *"a specific version of a package will always have one set of dependencies... There is one exception - packages with peer dependencies"*。

这直接解释了前端最著名的报错：

```text
Invalid hook call. Hooks can only be called inside of the body of a function component.
```

出现它时，**第一个要查的不是代码，是有没有两份 React**：

```bash
npm ls react      # 或 pnpm why react
# 看到 react@18.x 和 react@19.x 同时在树里 → 就是它
```

### 5. 失败模式对照：npm 硬失败，pnpm 只警告

实测同一个冲突（`react@18.3.1` + `react-dom@19.2.0`，react-dom 要求 `react@^19.2.0`）：

**npm：直接拒绝安装**，原文很长但值得记：

```text
npm error ERESOLVE unable to resolve dependency tree
npm error While resolving: peer-demo-2@1.0.0
npm error Found: react@18.3.1
npm error   react@"18.3.1" from the root project
npm error Could not resolve dependency:
npm error peer react@"^19.2.0" from react-dom@19.2.0
npm error Fix the upstream dependency conflict, or retry
npm error this command with --force or --legacy-peer-deps
npm error to accept an incorrect (and potentially broken) dependency resolution.
```

**pnpm：装成功，只打一条警告**，并提供了查询命令：

```text
[WARN] Issues with peer dependencies found. Run "pnpm peers check" to list them.

$ pnpm peers check
✕ unmet peer react
  Installed: 18.3.1
  Wanted:
    ^19.2.0:
      react-dom@19.2.0
```

| 维度 | npm | pnpm |
|---|---|---|
| peer 冲突时 | `ERESOLVE` 报错，装不下去 | 装下去，打 `WARN`（`strictPeerDependencies` 默认 `false`） |
| 结果安全性 | 逼你面对 | **需要你自觉**，`pnpm peers check` 不看就过去了 |
| 逃生舱 | `--legacy-peer-deps` | `strict-peer-dependencies=true` 反过来收紧 |

**这就是"pnpm 更严格"这句话最容易翻车的地方**：它严格在**模块解析**，宽松在**peer 校验**。`--legacy-peer-deps` 也不是解药——它等于退回 npm 6 的行为（不自动装 peer），可能换来实现里真的缺包。

### 6. 出问题了怎么办：一份排查与预防清单

```bash
# ① 定位：这个间接依赖到底从哪来的
pnpm why ms          # 实测输出：2.0.0 ← debug@2.6.9；2.1.3 ← 本项目，并提示 Found 2 versions of ms
npm ls react         # npm 侧同款，冲突时直接把链路画出来

# ② CI 必须用冻结安装，别让 lockfile 被悄悄改写
npm ci
pnpm install --frozen-lockfile
```

③ 强制统一版本：npm 用顶层 `overrides`，pnpm 写在 `pnpm-workspace.yaml`（下面这个位置有坑）。

pnpm 的 overrides 位置有个**静默坑**，实测（pnpm 11.5.2）：

```yaml
# ✅ pnpm-workspace.yaml —— 生效
overrides:
  ms: 2.0.0
# 实测结果：顶层 ms 变 2.0.0，.pnpm/ 里只剩 ms@2.0.0 一个版本
```

```json
// ❌ package.json 里的 pnpm.overrides —— 实测不生效，而且没有任何警告
// 顶层 ms 仍是 2.1.3，.pnpm/ 里 ms@2.0.0 和 ms@2.1.3 依然并存
{
  "pnpm": {
    "overrides": {
      "ms": "2.0.0"
    }
  }
}
```

```json
// npm 侧对应字段就是顶层 overrides（官方 package.json 文档列为标准字段）
{
  "overrides": {
    "ms": "2.0.0"
  }
}
```

最后补一条团队协作层面的：**`packageManager` 字段锁包管理器版本**（`"packageManager": "pnpm@11.5.2"`，配 Corepack 或 pnpm 的自管理版本能力），比在 README 里写"请用 pnpm 装"可靠得多。

## 其实你每天都在用

- **`node_modules` 里一堆你没装过的包**：那 67 个条目的 npm 项目就是它的日常形态。
- **新同事 clone 后 `pnpm i` 跑不起来**：多数是老项目里散落着幽灵依赖，临时 `shamefully-hoist=true` 能救急，但债还在。
- **CI 上 `npm ci` 报 ERESOLVE 而本地正常**：本地 `node_modules` 是历史遗留的树，`npm ci` 会先删掉重建，冲突这才现形。
- **`Invalid hook call`**：九成是 React 双实例，`npm ls react` 看一眼就知道。
- **两份 ESLint 插件互相不认**：`instanceof Plugin` 跨实例为 `false`，Symbol 也分裂，本质还是同一包多版本。
- **打包体积里同一个 lodash 出现两份**：`npm ls lodash` 顺着树找，用 overrides 收敛到一个版本。
- **lockfile 冲突删了重跑**：看起来干净，实际可能顺手升了一堆次要版本，改动混进 feature 分支，code review 时根本看不出来。
- **换了 pnpm 后 `postinstall` 的 patch 失效**：`patch-package` 依赖根 `node_modules` 的物理文件，pnpm 下建议改用内置的 `pnpm patch`。

## 常见误解（FAQ）

**❌ 误区 1：「pnpm 彻底消灭了幽灵依赖」**

只对**业务代码**成立。官方把默认布局叫 semistrict：依赖之间仍可通过 `node_modules/.pnpm/node_modules` 访问未声明的包（实测从 `accepts` 里能 resolve 到 `express`），而且那个隐藏层的目录名不带版本号，**同名多版本只留一个、留哪个不承诺**。准确说法是"项目代码不再有幽灵依赖"，而不是"整个项目没有幽灵依赖"。

**❌ 误区 2：「peerDependencies 只是提醒，不会真的安装」**

npm 3~6 是这样，npm 7 起默认会装（官方文档原话）。所以"只声明 react-dom，react 就自动来了"在 npm 下是事实。反过来说，如果某天它没自动来，问题通常在冲突而不是配置。

**❌ 误区 3：「peer 冲突就加 `--legacy-peer-deps`」**

那是把 npm 退回 v6 语义（不自动装 peer），属于"关掉检查"。它会同时关掉对**其他** peer 的自动安装，可能换来实现里真的缺包。正确顺序是：先 `npm ls` 看清冲突链路，能升就升，确实需要并存才用 overrides。

**❌ 误区 4：「`shamefully-hoist=true` 是兼容性问题的正解」**

它能一键解决"工具找不到包"，代价是把幽灵依赖请回根目录，等于放弃 pnpm 的核心收益。定位是**迁移期的临时桥**，不是长期配置；更细的替代是 `publicHoistPattern` 只放开确实需要的那几个包。

**❌ 误区 5：「lockfile 冲突，删掉重跑 `npm install` 就行」**

删掉即丢失全部版本约束，重装会按 `package.json` 的范围重新解析——你在修的是一次冲突，顺手改动了几十个包的版本。正确做法是保留 `main` 上的 lockfile，重新 `npm install` 生成一份再合并。

**❌ 误区 6：「`dependencies` 里的声明顺序决定谁被提升到顶层」**

本机实测（npm 10.9.7）：同一个依赖集合，把两个包的声明顺序反过来，得到的是**同一棵树**（顶层 `debug@4.4.3`，`finalhandler` 的 `debug@2.6.9` 一样嵌在 `finalhandler/node_modules` 里，lockfile 条目一致）。提升结果由 npm 的贪心遍历规则决定，官方从未承诺"最优"或"唯一"——**结论应该是别依赖具体布局，而不是赌某种顺序**。

**❌ 误区 7：「overrides 写在 package.json 里，pnpm 也认」**

实测 pnpm 11.5.2 下 `pnpm.overrides` 不生效且**不报任何错**（顶层版本没变、重复版本没被消除）。这是典型的静默失效——改完记得用 `pnpm why <包名>` 或看 `.pnpm/` 里还剩几个版本来自证。

## 一句话总结

**npm 把依赖提上来省了磁盘，代价是"未声明也能用"；pnpm 用符号链接把这条路在项目代码里堵死，但给依赖自己留了后门——真正要背的一条是：依赖解析出问题时先看树，别先改业务代码。**
