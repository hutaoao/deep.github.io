---
layout: post
title: "PWA 与 Service Worker 核心机制"
date: 2026-12-21 00:00:00 +0800
categories: ["网络与性能", "浏览器原理"]
tags: [PWA, "Service Worker", 离线缓存, 缓存策略, 生命周期]
description: >
  Service Worker 是浏览器里唯一能拦截网络请求的"可编程代理"。讲透它的生命周期、更新机制、
  fetch 与缓存策略，以及为什么用户总能看到旧版本页面。
---

## 一句话概括

PWA 不是一个框架，也不是一个打包工具，它是**三件套凑齐就算数**：

1. **Service Worker**——一个跑在独立线程、能拦截网络请求的"可编程代理"
2. **Web App Manifest**——一份 JSON，声明图标、名称、启动地址、显示模式
3. **HTTPS**——安全上下文，只有 `localhost` 例外

其中 Manifest 只决定"能不能装到桌面"，**离线能力完全由 Service Worker 的 fetch 处理负责**。

面试为什么爱问它？因为 Service Worker 是前端唯一一个"自己实现网络层"的机会。它的生命周期设计得很反直觉（新版本默认**不立即生效**），而线上的缓存事故（用户永远看到旧页面）几乎全部出自这里。

## 核心知识点

### 1. 先分清 Web Worker 和 Service Worker

两者名字像，面试常拿来挖坑。

| | Web Worker | Service Worker |
|---|---|---|
| 目标 | 把 CPU 密集任务挪出主线程 | 拦截网络请求，做离线/推送/后台同步 |
| 生命周期 | 跟着页面走，页面关了就没了 | **独立于页面**，没页面时也能被 push 唤起 |
| 控制范围 | 只有创建它的那个页面 | 该 origin 下**所有受控页面**（按 scope） |
| DOM | 没有 | 也没有 |
| 网络 | 能发请求，**不能拦截** | **能拦截所有 fetch** |
| 常用 API | `postMessage`、TypedArray | `Cache`、`FetchEvent`、`Push`、`Background Sync` |
| 是否长期驻留 | 不能 | 会被浏览器随时终止，靠事件唤醒 |

一句话记住：**Web Worker 是"另一个线程"，Service Worker 是"另一层网络"。**

### 2. 它在什么环境里跑：一堆"没有"

Service Worker 跑在 worker 上下文里，所以下面这些**都没有**（面试很爱问）：

- 没有 `window`、没有 DOM，不能操作页面
- **不能用 `localStorage` / `sessionStorage`**，也不能用同步 XHR——worker 设计上是全异步的
- **不能用动态 `import()`**：在 service worker 全局作用域里调用会直接抛错；静态 `import` 是可以的
- 只能用 `self` 代表自己；存储走 **Cache API** 和 **IndexedDB**

另外调试上有个容易懵的点：Service Worker 的日志不一定出现在页面控制台里，要看它自己的调试面板。Chrome 上是 `chrome://inspect/#service-workers` 点 inspect，或者 DevTools 的 Application → Service Workers → inspect（MDN 里也是这么推荐的）。

### 3. 生命周期：为什么新版本默认"待机"

这五个状态是必背的：

```text
parsed → installing → installed(waiting) → activating → activated
                                  ↘ redundant（被替换 / 安装失败）
```

完整走一遍：

1. **下载**：`navigator.serviceWorker.register('/sw.js')`，浏览器去拉脚本
2. **install**：触发 `install` 事件。标准动作是**预缓存**关键资源（App Shell），用 `event.waitUntil(...)` 把这件事挂到生命周期上。**如果这个 Promise reject，整个安装失败、SW 被丢弃，旧的继续用**——这就是"预缓存要么全成功、要么不上线"的保证
3. **installed / waiting**：如果页面里已有旧 SW 在跑，新 SW 就停在这里待机，**只有所有旧页面都关闭后才会激活**。这是"改了 `sw.js` 用户还是旧版本"的第一层原因，也是设计意图：避免新 SW 去处理旧页面的请求，造成版本错配
4. **activate**：触发 `activate` 事件，标准动作是**清理旧缓存**（遍历 `caches.keys()`，删掉不在白名单里的），同样要 `waitUntil`
5. **activated**：开始处理 `fetch`、`push` 等事件

```js
const CACHE = 'app-shell-v3';                       // 缓存名带版本号

self.addEventListener('install', (event) => {
  event.waitUntil(caches.open(CACHE).then((c) => c.addAll(SHELL)));
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys()
      .then((keys) => Promise.all(
        keys.filter((k) => k !== CACHE).map((k) => caches.delete(k))
      ))
      .then(() => self.clients.claim())              // 首次注册时立刻接管
  );
});
```

两个"提前"的开关，必须连着代价一起说：

- `self.skipWaiting()`：让 waiting 中的新 SW **立刻激活**，不等旧页面关闭
- `self.clients.claim()`：让刚激活的 SW **立刻接管已打开的页面**，不然要等下一次导航

**`skipWaiting` 是危险动作**：旧页面还在跑老代码，新 SW 已经在用新缓存策略响应它的请求，页面资源和 SW 逻辑版本错配，可能直接白屏。生产上的正确姿势是：监听到 `updatefound` 之后**弹一个"有新版本，点此刷新"的提示**，用户确认了再 `skipWaiting` + `clients.claim()`。

还有一个反直觉点：**首次注册后，当前页面是不受 SW 控制的**——第一次访问时 `fetch` 事件根本不会触发，刷新一次才生效。想让当次就生效，才需要 `clients.claim()`。

### 4. 更新机制：浏览器怎么判断"你改了"

面试高频："**我改了 `sw.js`，为什么用户还是旧版本？**"答案有三层，得都答到。

**第一层：什么时候去检查有没有新 SW**

- 用户**导航到 scope 范围内的页面**时
- Service Worker 上触发了功能性事件（比如 push），且**距上次下载已超过 24 小时**
- 手动调 `registration.update()`

**第二层：怎么判断"新"**

浏览器把拉到的 `sw.js` 和现有的做**字节级比较**——只要有一个字节不同就算新版本（所以改个注释里的日期也算），然后触发 install。

**第三层：新 SW 卡在 waiting**

就是上面说的，旧页面不关闭，新 SW 不激活。

除了这三层，还有一个**极隐蔽的坑**：

```js
// sw.js 里
importScripts('/lib/workbox.js');   // 这个文件不会跟着 sw.js 一起更新
```

`updateViaCache` 的默认值是 `'imports'`，含义是：**`sw.js` 本身不走 HTTP 缓存，但用 `importScripts()` 导入的脚本会走 HTTP 缓存**。结果就是：你改了 `workbox.js`，浏览器拿的还是缓存里的旧版本，而 `sw.js` 一个字节都没变，**更新压根没触发**。两种正经解法：把依赖直接打进 `sw.js`（用构建工具内联），或者给 `importScripts` 的地址带上版本号。

**"用户看到旧页面"的三个独立原因**（排查口诀）：

1. 新 SW 还卡在 waiting 状态（代码/交互问题）
2. `sw.js` 本身被 HTTP 缓存或 CDN 缓存了（部署问题）
3. 缓存名没升级，fetch 策略一直命中旧 Cache（缓存策略问题）

很多团队只排查第 1 条，然后开始怀疑浏览器有 bug。

### 5. fetch 事件与五种缓存策略

`fetch` 事件是 Service Worker 的核心：**受控页面发出的每个请求都会先到它这里**，用 `event.respondWith(responseOrPromise)` 决定返回什么。

几条硬规则（来自 MDN，面试可以直接引）：

- `respondWith` **必须在事件分发阶段同步调用**，且**一个事件只能调一次**，否则抛 `InvalidStateError`
- 返回的 `Response.type` 和 `request.mode` 必须匹配（比如 `no-cors` 的请求只能回 `opaque` 响应），否则抛 `NetworkError`
- 返回的 `Response.url` 会作为最终 URL 传播（Firefox 59+ 起），这会影响被拦截的样式表里相对 `@import` 的解析基准

**不调 `respondWith` 会怎样？** 等于没拦截，浏览器按默认方式走网络。所以"离线打不开"的一个常见原因，就是 fetch 处理器里某个分支忘了 `return`。

五种策略，这张表要背：

| 策略 | 逻辑 | 适合 | 代价 |
|---|---|---|---|
| Cache First | 先查缓存，没有才走网络并写入 | 带 hash 的静态资源、字体、图标 | 永不自动更新 |
| Network First | 先走网络，失败回退缓存 | 接口数据、需要新鲜的 HTML | 慢、离线体验差 |
| Stale While Revalidate | **立刻返回缓存**，同时后台请求刷新缓存 | 非关键资源（图片、非首屏 JS） | 仍会读到一次旧值 |
| Network Only | 完全不碰缓存 | 埋点、POST 等非幂等请求 | 离线直接失败 |
| Cache Only | 只读缓存，没有就失败 | 已预缓存的 App Shell | 必须自己保证命中 |

工程上一般**按请求类型分流**，这也是 Workbox 默认套路的思路：

```js
self.addEventListener('fetch', (event) => {
  const { request } = event;
  if (request.method !== 'GET') return;                        // 非 GET 直接放行

  if (request.mode === 'navigate') {                           // 页面导航：先网络
    event.respondWith(
      fetch(request).catch(() => caches.match('/offline.html'))
    );
    return;
  }
  // 静态资源：先缓存
  event.respondWith(
    caches.match(request).then((hit) => hit || fetch(request))
  );
});
```

### 6. Cache API 的三个反直觉点 + 一个预缓存大坑

**反直觉点一：Cache API 不遵守 HTTP 缓存头。** MDN 原文：*"The caching API doesn't honor HTTP caching headers."* 你设的 `Cache-Control: max-age=3600` 对 `caches` 里的副本完全无效。

**反直觉点二：缓存不会自动过期。** *"Items in a Cache do not get updated unless explicitly requested; they don't expire unless deleted."* 所以清理必须自己在 `activate` 里做，缓存名必须带版本号。

**反直觉点三：匹配是由"键 + `Vary` 头"共同决定的。** 缓存命中的算法会参考响应里的 `Vary` 头，同一个 URL 可能因为请求头不同而匹配到不同条目——给接口缓存开了 `Vary`，命中率会莫名其妙地变低。

**预缓存的大坑：`cache.addAll()` 遇到 opaque 响应会整体失败。**

MDN 明确列了 `addAll` 抛 `TypeError` 的两种情形：

- URL 的 scheme 不是 `http` 或 `https`
- **响应状态码不在 2xx 范围**——而且特别点出"请求是**跨域 no-cors** 时状态码恒为 0"

后半句就是陷阱：**跨域 CDN 上的字体、第三方脚本，如果没带 CORS 头，拿到的是 opaque 响应（status 为 0、内容不可读），`addAll` 会直接 reject**。再加上 `addAll` 的原子性——要么全成、要么全不成——一个 CDN 抖一下，你的 App Shell 就整个没法预缓存。

所以正确做法是：

- 预缓存列表里**只放同源或已开 CORS 的关键资源**，保持精简
- 跨域资源要么让 CDN 返回 `Access-Control-Allow-Origin`；要么改用 `cache.put()` + 手动 `fetch()`（`put` 是允许存 opaque 响应的），但要接受**opaque 响应不可读**，你无法判断它其实是个 404
- 另外注意：同一次 `addAll` 的列表里**不能有重复 URL**，否则 put 互相覆盖也会失败

### 7. 作用域与部署：三个能直接踩到线上的点

- **scope 默认是 `sw.js` 所在目录**（把 `./` 相对 `scriptURL` 解析）。想让 SW 控制整个站点，`sw.js` 必须放在根目录，或用响应头 `Service-Worker-Allowed` 显式放宽。放在 `/js/sw.js`，它只管得了 `/js/` 下面的请求
- **一个文档可能同时落在多个注册的 scope 里**，这时浏览器按**最具体（最长）的那个 scope** 匹配，保证一个文档只有一个 SW 在跑；所以别设计互相重叠的 scope
- **SW 脚本必须以合法的 JS MIME 类型返回**（MDN 原文要求），返回 `text/html` 或 `application/octet-stream` 会导致注册失败——部署时 Content-Type 配错是最常见的原因
- **HTTPS 是硬性要求**，只有 `localhost` 例外；否则注册直接抛 `SecurityError`

## 其实你每天都在用

- 手机上把某个网站"添加到主屏幕"，图标和启动页都像 App——Manifest 在起作用
- 断网状态下打开某个网页，还能看到上次的内容和一句"你已离线"——App Shell + 离线兜底页
- 部署完之后同事说"我这还是旧页面"，两个人强刷的结果还不一样——SW 缓存
- Chrome DevTools → Application → Service Workers 里那个 **"Update on reload"** 勾选框，开发时几乎人人都点过
- 改完 `sw.js` 本地怎么都不生效，最后发现要**把所有标签页都关掉**——waiting 状态
- Lighthouse 跑分里的"可安装性"审计，要求的正是 Manifest + SW + HTTPS 三件套
- 网站弹的"有新版本，点击刷新"浮层——`updatefound` 事件 + 用户确认后才 `skipWaiting` 的标准实现

## 常见误解（FAQ）

**❌ 误区1："Service Worker 就是 Web Worker 的一种，用法差不多"**
它们共享 worker 的 API 基础（`postMessage`、`importScripts`），但目的、生命周期、控制范围完全不同：Web Worker 是一个计算线程，Service Worker 是一层网络代理，还能在页面全关的情况下被 push 唤起。面试答"就是个后台线程"会被一路追问到崩。

**❌ 误区2："Service Worker 里可以用 localStorage 存点东西"**
不行。Web Storage 是同步 API，worker 上下文里不可用（同步 XHR 同理）。SW 里的存储只有 **Cache API** 和 **IndexedDB**。

**❌ 误区3："注册了 Service Worker，页面马上就被控制了"**
首次注册时当前页面不受控，要等下一次导航才生效，所以第一次访问时 `fetch` 事件不会触发。要当次就接管，得在 `activate` 里调 `clients.claim()`。

**❌ 误区4："改了 `sw.js` 用户立刻就能看到新版本"**
四道关卡：能不能触发更新检查（导航 / 24 小时 / 手动 `update()`）、字节比对是否判定为新、新 SW 能不能越过 waiting、以及 `sw.js` 本身有没有被 HTTP 或 CDN 缓存。任何一道没过，用户看到的都是旧版本。

**❌ 误区5："给资源加上 `Cache-Control`，Cache API 就会按它过期"**
不会。Cache API **不遵守 HTTP 缓存头**，缓存条目也不会自动过期，必须自己在 `activate` 里按缓存名版本号清理。把 Cache API 当成"浏览器 HTTP 缓存"来理解，是很多缓存事故的起点。

**❌ 误区6："`cache.addAll` 可以把 CDN 上的字体一起预缓存"**
跨域且没有 CORS 头的资源拿到的是 **opaque 响应（status 0）**，`addAll` 判定状态码不在 2xx 会直接 reject，而它是**原子操作**——整个预缓存跟着一起失败。要么让 CDN 开 CORS，要么改用 `cache.put()` 手动存，但要接受 opaque 响应不可读。

**❌ 误区7："做 PWA 就是缓存所有资源，越全越好"**
预缓存追求的是**最小可用集合**（App Shell）：列表越大，安装越慢、失败概率越高、存储占用越大。运行时该按需缓存的交给 fetch 策略，别一股脑塞进 `install`。

## 一句话总结

Service Worker 的全部复杂度都来自一个设计选择——**它是"独立于页面、又会被随时终止的网络代理"**。记住这一点，生命周期为什么要 waiting、缓存为什么要手动清、缓存名为什么必须带版本号，就都能自己推出来了。
