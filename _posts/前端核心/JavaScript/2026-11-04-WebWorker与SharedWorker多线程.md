---
layout: post
title: "Web Worker 与 SharedWorker 多线程"
date: 2026-11-04 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [Web Worker, SharedWorker, postMessage, 结构化克隆, Transferable, 多线程]
description: >
  Worker 到底能干什么不能干什么、postMessage 的结构化克隆开销与 Transferable 零拷贝、SharedArrayBuffer 的跨源隔离前提，以及 Worker / SharedWorker / ServiceWorker 三兄弟怎么选。
---

## 一句话概括

JS 是单线程的，一段 500ms 的同步计算就能让页面完全卡死——按钮点不动、动画掉帧。**Web Worker 就是浏览器给你的"真后台线程"**：把重活丢过去算，主线程继续响应用户。

面试问它，考的不是"你会不会 new 一个 Worker"（那两行代码谁都会写），而是三件事：**Worker 里哪些 API 不能用**、**主线程和 Worker 之间的数据是怎么传的（拷贝还是共享，代价多大）**、**Worker / SharedWorker / ServiceWorker 三个东西的边界在哪**。把这三点讲清楚就过关了。

## 核心知识点

### 1. Worker 里没有 DOM——这是它最大的限制

**结论：Worker 有自己独立的全局作用域，拿不到 `window`、`document`、也读不到主线程的变量。** 它的全局对象叫 `self`（`DedicatedWorkerGlobalScope`）。

能用的 / 不能用的，面试直接背这张表：

| Worker 里能用 | Worker 里没有 |
|---|---|
| `fetch`、`XMLHttpRequest`、`WebSocket` | `document` / DOM 操作 |
| `setTimeout` / `setInterval` | `window`、`alert` |
| `IndexedDB`、`Cache` | `localStorage` / `sessionStorage` |
| `crypto`、`TextEncoder`、`OffscreenCanvas` | `requestIdleCallback` |
| `importScripts()`（仅 classic worker） | 主线程的任何变量 |

```js
// worker.js —— 全局是 self，不是 window
self.onmessage = (e) => {
  const result = heavyCompute(e.data);
  self.postMessage(result); // 只能把结果发回去，不能自己改 DOM
};
```

> 注意 `localStorage` 在 Worker 里是拿不到的（它是同步 API，规范上不暴露给 Worker），要持久化只能用 `IndexedDB`。这个细节问出来的频率还挺高。

### 2. 创建 Worker：现代写法一定用 `new URL` + module

**结论：写死字符串路径的老写法会被打包工具搞坏，正确姿势是 `new URL('./x.js', import.meta.url)` 配 `{ type: 'module' }`。**

原因很直接：Vite / webpack / Rspack 都专门识别这个模式，会把 worker 单独打成一个 chunk 并正确处理 hash 后的文件名；而裸字符串它们只能当普通静态资源，构建后 404 是家常事。

```js
// ❌ 构建后路径可能失效，且 worker 内不能用 import
const w1 = new Worker('./worker.js');

// ✅ 打包工具能识别，worker 内可直接用 ESM import
const w2 = new Worker(new URL('./worker.js', import.meta.url), { type: 'module' });
```

两个坑要记住：`new URL()` 必须**直接写在** `new Worker()` 里（提取成变量就识别不到了），并且 options 里的值必须是字面量。`type: 'module'` 之后 worker 里就能用 `import`，`importScripts()` 属于 classic worker 的老写法，新代码别用了。

### 3. postMessage 是"拷贝"，不是"共享"

**结论：`postMessage` 走的是结构化克隆算法（structured clone），数据被深拷贝一份过去，两边改各自的互不影响。**

这一条是面试最爱追问的点，配套两个结论：

- **能传什么**：原始值、普通对象、数组、`Map`/`Set`、`Date`、`RegExp`、`ArrayBuffer`、`Blob`、`ImageData`，甚至支持循环引用。
- **不能传什么**：函数、DOM 节点、Symbol 作为键的属性、带原型方法的类实例（方法会丢），传了直接抛 `DataCloneError`。

```js
// ❌ 函数不能被克隆，直接 DataCloneError
worker.postMessage({ done: () => console.log('ok') });

// ✅ 只传纯数据，行为用约定的 type 字段表达
worker.postMessage({ type: 'parse', payload: rawText });
```

还有一个工程细节：`postMessage` **一次只能发一个对象**，多个值要包成对象或数组。而且 Worker 的消息是"广播式"的，没有天然的请求-响应配对，多任务并发时必须自己带 id：

```js
// 主线程：用 id 把响应和请求对上，否则并发时结果会串
const pending = new Map();
worker.onmessage = ({ data }) => pending.get(data.id)?.(data.result);

function run(payload) {
  const id = crypto.randomUUID();
  return new Promise((resolve) => {
    pending.set(id, resolve);
    worker.postMessage({ id, payload });
  });
}
```

### 4. 大数据别拷贝：Transferable 零拷贝转移

**结论：克隆 10MB 的 buffer 要花好几毫秒还翻倍占内存；把它放进 `postMessage` 的第二个参数（transfer 列表），就变成"移交所有权"的零拷贝操作。**

代价是：**转移后发送方就再也不能用了**（对象被 detached，`byteLength` 变 0，再读写抛异常）。

```js
const buf = new ArrayBuffer(10 * 1024 * 1024);

// ❌ 默认克隆，10MB 复制一遍，两个线程各占一份内存
worker.postMessage({ buf });

// ✅ 转移所有权，O(1) 交接
worker.postMessage({ buf }, [buf]);
console.log(buf.byteLength); // 0 —— 主线程这边已经不能用了
```

一个非常容易错的细节：**TypedArray（`Uint8Array` 等）本身不可转移，可转移的是它底层的 `ArrayBuffer`**。所以要写成：

```js
// ✅ transfer 列表里放 .buffer，不是 TypedArray 本身
worker.postMessage({ samples: float32Arr }, [float32Arr.buffer]);
```

常见的可转移对象：`ArrayBuffer`、`MessagePort`、`ImageBitmap`、`OffscreenCanvas`、`ReadableStream` / `WritableStream` / `TransformStream`、`VideoFrame`、`AudioData`。

> 面试加分点：**忘记 transfer 不会报错，只会悄悄变慢**。所以性能排查时看到"很小的消息也要几毫秒"，就该怀疑是在克隆大 buffer。

### 5. SharedArrayBuffer：真共享内存，但有前置条件

**结论：`SharedArrayBuffer` 是两个线程同时看到同一块内存，不拷贝也不转移；但它要求页面开启"跨源隔离"，否则 `SharedArrayBuffer` 直接是 `undefined`。**

需要服务端下发两个响应头（Spectre 漏洞之后加的硬性要求）：

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

而且多线程同时读写同一块内存，必须用 `Atomics` 做同步（`Atomics.add` / `compareExchange` / `wait` / `notify`），否则结果是未定义的。

```js
// ✅ 上线前先做能力检测，缺头就降级回 postMessage
if (typeof SharedArrayBuffer === 'undefined') {
  console.warn('未跨源隔离，退回克隆方案');
}
```

实际项目里 90% 的场景 transfer 就够了，`SharedArrayBuffer` 只在高频共享状态（如 WASM 多线程、音频处理）才值得上。面试能说清"它要 COOP/COEP 头"就已经比大多数人强了。

### 6. Worker 不便宜：用池子，别一个任务 new 一个

**结论：创建一个 Worker 有几毫秒的启动开销，还要单独占一份 JS 堆内存，所以高频任务要用 Worker 池复用。**

池子大小一般取 `navigator.hardwareConcurrency - 1`（留一个核给主线程），任务用轮询派发：

```js
// 核心思路：固定几个 worker 轮流用，避免反复创建销毁
const size = Math.max(1, (navigator.hardwareConcurrency || 4) - 1);
const pool = Array.from({ length: size }, () =>
  new Worker(new URL('./w.js', import.meta.url), { type: 'module' })
);
let next = 0;
const pick = () => { const w = pool[next]; next = (next + 1) % pool.length; return w; };
```

用完记得 `worker.terminate()` 释放。注意 `terminate()` 是**立即杀死、没有清理回调**的，需要落盘的状态得提前处理好。

### 7. Worker / SharedWorker / ServiceWorker 三兄弟对比

这是最经典的一道对比题，一张表说完：

| | Web Worker（专用） | SharedWorker（共享） | Service Worker |
|---|---|---|---|
| 定位 | 单页面的计算苦力 | 同源多标签页共享一个线程 | 网络代理 / 离线缓存 |
| 通信 | `worker.postMessage` 直连 | 必须走 `port` 端口 | postMessage / Cache / IndexedDB |
| 生命周期 | 跟着创建它的页面走 | 所有同源页面都关了才销毁 | 浏览器托管，页面关了还活着 |
| 能拦请求吗 | 不能 | 不能 | **能**（`fetch` 事件） |
| HTTPS | 不要求 | 不要求 | **生产必须**（localhost 例外） |
| 典型场景 | 大文件解析、加解密、图像处理 | 多标签页共享一条 WebSocket | PWA 离线、接口缓存、推送 |

一句话记法：**Worker 干活，SharedWorker 干活且多个标签页共用，ServiceWorker 管网络。**

### 8. SharedWorker 的 port 模型和那个必踩的坑

**结论：SharedWorker 不直接 `postMessage`，得通过 `worker.port`；而且用 `addEventListener('message')` 时必须手动调 `port.start()`，用 `port.onmessage = fn` 时才会隐式帮你调。**

这个坑非常经典——消息发不出去、收不到，八成就是漏了 `start()`：

```js
// 主线程
const sw = new SharedWorker(new URL('./shared.js', import.meta.url));

// ❌ 用 addEventListener 但没 start()，消息队列不会流动
sw.port.addEventListener('message', onMsg);

// ✅ 补上 start()
sw.port.addEventListener('message', onMsg);
sw.port.start();

// ✅ 或者直接用 onmessage，隐式 start
sw.port.onmessage = onMsg;
```

SharedWorker 内部靠 `connect` 事件拿到每个页面的 port，想做多标签页广播就自己维护一个 port 列表：

```js
// shared.js —— 收集所有连接进来的端口，实现跨标签页广播
const ports = [];
self.onconnect = (e) => {
  const port = e.ports[0];
  ports.push(port);
  port.onmessage = (ev) => ports.forEach((p) => p.postMessage(ev.data));
};
```

**移动端是它的硬伤**：Android 上的 Chrome / WebView 长期不支持 SharedWorker（MDN 兼容表长期标 No），Safari 也是中间断供多年、16 之后才回来。所以纯粹只做"多标签页同步状态"，用 `BroadcastChannel` 更省心；只有确实需要"多标签页共享一条连接 / 一份计算结果"时才上 SharedWorker，并且务必做能力检测降级。

### 9. Worker 里报错不会拖垮主线程，但会静默消失

**结论：Worker 内未捕获的错误会在主线程触发 `worker.onerror`，主线程本身不崩；但你不监听，这个错就彻底没了。**

```js
// ✅ onerror 必须挂，否则 worker 里的异常你永远不知道
worker.onerror = (e) => {
  console.error('worker 挂了：', e.message, e.filename, e.lineno);
  worker.terminate();
};
```

## 其实你每天都在用

- Figma 把布局计算放进 Worker，所以画布拖动还能稳在 60fps。
- 在线 Excel / CSV 导入时前端解析几十万行不卡，基本都是 Worker 干的。
- 图片编辑类网页做 4K 滤镜，用 Worker + `OffscreenCanvas` 在后台逐像素处理。
- 前端加解密、大 JSON diff、代码高亮预处理，都是典型的丢给 Worker 的活。
- Comlink 这个库把 Worker 包装成"像调用普通 async 函数"，用过 Vite 生态的大概率见过。
- 语雀 / 飞书这类多标签页应用同步登录态，用的是 `BroadcastChannel` 或 SharedWorker 那一套思路。

## 常见误解（FAQ）

**❌ 误区1："Worker 能让代码跑得更快。"**
不对。计算量一点没少，耗时几乎一样。Worker 换来的是**主线程的响应性**——用户还能点、动画还能动。要真提速得靠 Worker 池并行拆分，那是另一回事。

**❌ 误区2："postMessage 传对象是传引用，改了两边都变。"**
错得很彻底。它是结构化克隆，深拷贝一份。主线程改了原对象，Worker 里那份完全不受影响，反之也一样。

**❌ 误区3："Worker 里可以操作 DOM，只是不推荐。"**
不是不推荐，是**根本拿不到 `document`**。想更新界面只有一条路：把结果 `postMessage` 回主线程，由主线程改 DOM。

**❌ 误区4："SharedWorker 和 Web Worker 就差个名字。"**
差的是通信模型和生命周期：SharedWorker 必须走 `port`、要考虑 `start()`、生命周期跟"所有同源页面"绑定，而且移动端支持很差。

**❌ 误区5："传大数组用 transfer 更快，那所有消息都加 transfer 好了。"**
不行。转移之后发送方对象就废了，下一帧你还想复用这个 buffer 就得重新分配。高频场景一般配一个 buffer 池来回倒。

**❌ 误区6："SharedArrayBuffer 直接 new 就能用。"**
没开跨源隔离（COOP + COEP 两个响应头）的话，它在浏览器里是 `undefined`。而且有些 CDN 会把这两个头剥掉，得在运行时检测。

## 一句话总结

**Worker 不让代码变快，只让主线程不卡；数据默认是克隆传递，大 buffer 要 transfer 零拷贝，真共享内存得先开跨源隔离——Worker 干活、SharedWorker 多标签页共用、ServiceWorker 管网络。**
