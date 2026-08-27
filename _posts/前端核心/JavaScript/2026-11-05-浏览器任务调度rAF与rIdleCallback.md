---
layout: post
title: "浏览器任务调度：rAF 与 rIdleCallback"
date: 2026-11-05 00:00:00 +0800
categories: ["前端核心", "JavaScript"]
tags: [requestAnimationFrame, requestIdleCallback, 任务调度, 事件循环, INP, scheduler]
description: >
  一帧里 rAF 和 rIdleCallback 分别在什么位置执行、为什么动画必须用 rAF、rIC 的三条禁忌和兼容性短板，以及现在优化 INP 更该用的 scheduler.postTask / scheduler.yield。
---

## 一句话概括

浏览器主线程只有一个，JS 执行、样式计算、布局、绘制、响应输入全挤在上面。**任务调度要解决的问题就一句话：什么活该在这一帧里干，什么活该往后挪。**

面试问 `requestAnimationFrame` 和 `requestIdleCallback`，本质是在问你**懂不懂一帧的时序**。rAF 是"下次绘制前给我一次机会"，rIC 是"这帧绘制完了还有空的话再叫我"。一个是高优先级的视觉任务，一个是低优先级的后台任务，位置差一步，用途完全不同。

再往下问一层，就会问到 2026 年更该用的东西：`scheduler.postTask` 和 `scheduler.yield`——**这两个才是现在优化 INP 的正牌工具**。

## 核心知识点

### 1. 先把一帧的时序背下来

**结论：一帧内的顺序是「宏任务 → 微任务 → rAF 回调 → 样式计算/布局/绘制 → 如果还有剩余时间，执行 rIC 回调」。**

这个顺序是所有结论的根，记住它其他都能推出来：

| 阶段 | 跑什么 | 特点 |
|---|---|---|
| ① 宏任务 | `setTimeout`、事件回调、网络回调 | 一次事件循环取一个 |
| ② 微任务 | `Promise.then`、`queueMicrotask` | 清空为止，会插队 |
| ③ rAF 回调 | `requestAnimationFrame` | **在样式/布局之前**，改 DOM 最划算 |
| ④ 渲染 | 样式计算 → 布局 → 绘制 → 合成 | 与屏幕刷新（vsync）同步 |
| ⑤ 空闲期 | `requestIdleCallback` | 帧还有余量才跑，可能一直不跑 |

一句话记法：**rAF 在画之前，rIC 在画之后。**

### 2. rAF：跟着屏幕刷新率走，动画只能用它

**结论：rAF 的回调在下一次重绘前执行，频率跟显示器刷新率对齐（60Hz 约 16.7ms 一次，120Hz 就是 8.3ms 一次），所以动画天然不掉帧。**

三个必须知道的细节：

1. 回调参数是一个 `DOMHighResTimeStamp`（等价于 `performance.now()`），**同一帧内所有 rAF 回调收到的时间戳是同一个值**，用它算 delta 才准。
2. 页面切到后台 / 标签页隐藏时，rAF **自动暂停**，省电省 CPU。
3. 它只保证执行一次，要连续动画必须在回调里递归再调一次。

```js
// ✅ 用回调传进来的时间戳算位移，不同刷新率下速度一致
let start;
function step(now) {
  start ??= now;
  const p = Math.min((now - start) / 300, 1); // 300ms 走完
  box.style.transform = `translateX(${p * 200}px)`;
  if (p < 1) requestAnimationFrame(step);
}
requestAnimationFrame(step);
```

对比 `setTimeout` 做动画为什么不行：

```js
// ❌ 定时器和 vsync 不同步，16ms 也会漂移，掉帧、撕裂
setInterval(() => (box.style.left = x++ + 'px'), 16);

// ✅ 跟着帧走
requestAnimationFrame(function loop() { box.style.left = x++ + 'px'; requestAnimationFrame(loop); });
```

取消用 `cancelAnimationFrame(id)`。组件卸载不取消是最常见的内存泄漏源之一：

```js
// ✅ React 里必须存 id 并在 cleanup 里取消
useEffect(() => {
  const id = requestAnimationFrame(step);
  return () => cancelAnimationFrame(id);
}, []);
```

### 3. rAF 的两个实战用法（面试很爱问）

**滚动/resize 节流**：滚动事件触发频率远高于帧率，直接改样式等于白算。用 rAF 把它压到"每帧最多一次"：

```js
// ✅ 一帧内只更新一次，多余的 scroll 事件直接丢
let ticking = false;
addEventListener('scroll', () => {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => { updateHeader(); ticking = false; });
});
```

**"双 rAF"让过渡动画生效**：新插入的元素同一帧内改 class，浏览器还没算过初始样式，transition 不会触发。等一帧让样式落定，再改：

```js
// ✅ 等浏览器完成一次渲染，下一帧再切 class，transition 才有起点
el.classList.add('enter');
requestAnimationFrame(() => requestAnimationFrame(() => el.classList.add('enter-active')));
```

### 4. rIC：把不着急的活塞进帧的缝隙里

**结论：`requestIdleCallback(cb, { timeout })` 只在浏览器这一帧干完活还有余量时执行，回调收到一个 `IdleDeadline` 对象，靠 `deadline.timeRemaining()` 判断还剩多少毫秒。**

两个关键属性：

- `timeRemaining()`：剩余空闲毫秒数，**上限就是 50ms**，实际经常只有几毫秒；空闲期结束返回 0。
- `didTimeout`：为 `true` 表示"不是因为空闲，而是你设的 `timeout` 到了被强制执行"，此时 `timeRemaining()` 约等于 0。

标准的分片写法长这样：

```js
// ✅ 有多少时间干多少活，干不完下次接着来
function work(deadline) {
  while ((deadline.timeRemaining() > 1 || deadline.didTimeout) && queue.length) {
    handle(queue.shift());
  }
  if (queue.length) requestIdleCallback(work, { timeout: 2000 });
}
requestIdleCallback(work, { timeout: 2000 });
```

```js
// ❌ 不看 deadline，一口气跑 100ms，下一帧直接掉帧
requestIdleCallback(() => heavyLoop());
```

`timeout` 是**防饿死的兜底**：页面一直很忙（游戏循环、广告脚本）就可能永远没有空闲期，加了 timeout 至少保证会被执行。取消用 `cancelIdleCallback(id)`。多个 rIC 回调按 FIFO 顺序执行。

### 5. rIC 的三条禁忌

这几条是 MDN 明确写的，面试说出来很加分：

**① 别在 rIC 里改 DOM。** 它执行时这一帧的布局和绘制都已经完成了，你改 DOM 会让浏览器额外做一次重排，甚至引起布局抖动。要改 DOM 就在 rIC 里算好数据，然后用 rAF 去落地。

```js
// ❌ 在 idle 回调里直接写 DOM，制造额外重排
requestIdleCallback(() => { list.innerHTML = render(data); });

// ✅ idle 算数据，rAF 改 DOM
requestIdleCallback(() => { const html = render(data); requestAnimationFrame(() => (list.innerHTML = html)); });
```

**② 别在 rIC 里 resolve / reject Promise。** 因为回调一返回，那个 Promise 的 then 就会作为微任务立刻执行，等于把不可控的耗时又塞回了这一帧。

**③ Worker 里没有 rIC。** Background Tasks API 不暴露给 Web Worker，`self.requestIdleCallback` 是 undefined。

### 6. 兼容性：rIC 至今不是 Baseline

**结论：`requestIdleCallback` 在 Safari 上一直没正式支持（桌面版长期只在 TP 里带开关，iOS Safari 直接不支持），MDN 上明确标注它"不是 Baseline"。**

这意味着你不能裸用，必须降级。最简单的兜底：

```js
// ✅ 特性检测 + setTimeout 降级（注意 deadline 要伪造出来）
const rIC = globalThis.requestIdleCallback ||
  ((cb) => setTimeout(() => cb({ timeRemaining: () => 5, didTimeout: false }), 1));
```

Chrome 47+、Firefox 55+、Edge 79+ 是支持的，所以内部后台系统用它没问题，面向 C 端（尤其 iOS 占比高）必须做降级。

### 7. 更现代的答案：scheduler.postTask 与 scheduler.yield

这是能明显区分"背过八股"和"真在优化性能"的地方。

**`scheduler.postTask(cb, { priority })` 支持三档优先级**，比 rIC 只有"空闲"一档精细得多：

| 优先级 | 用途 |
|---|---|
| `user-blocking` | 最高，直接阻塞用户交互的关键活 |
| `user-visible` | 默认，影响用户所见但不阻塞 |
| `background` | 最低，埋点上报、预取、清理 |

```js
// ✅ 关键渲染优先，埋点丢到后台档
if (globalThis.scheduler?.postTask) {
  scheduler.postTask(renderUI, { priority: 'user-blocking' });
  scheduler.postTask(sendAnalytics, { priority: 'background' });
} else {
  setTimeout(renderUI, 0); // Safari 降级
}
```

它返回 Promise，可以 `await`，也能配 `TaskController` / `AbortSignal` 取消。

**`scheduler.yield()` 是现在拆长任务、优化 INP 的首选**。它在循环中间把控制权交还浏览器，让浏览器有机会响应用户输入，然后继续跑：

```js
// ✅ 每处理一项就让一次，长任务被切碎，INP 明显改善
for (const item of items) {
  doWork(item);
  await scheduler.yield();
}
```

它比 `await new Promise(r => setTimeout(r, 0))` 好在哪？**`setTimeout` 让出去之后，你的续作会被排到队列最后面**，前面插进来的任务都得先跑完；而 `scheduler.yield()` 的续作是优先恢复的，"让位但不失去位置"。

支持情况要说清楚：`postTask` 是 Chrome/Edge 94+、Firefox 142+；`yield` 是 Chrome/Edge 129+、Firefox 142+；**Safari 两个都还不支持**，所以必须写特性检测 + `setTimeout` 兜底。

> 特别注意：`postTask` / `yield` **都不会把代码移出主线程**，它们只是排队方式更聪明。真要并行还得靠 Web Worker。

### 8. 五种调度手段横向对比

面试让你"把这些 API 排个序"，直接给这张表：

| API | 执行时机 | 典型用途 |
|---|---|---|
| `queueMicrotask` | 当前任务结束、渲染之前 | 状态同步，最紧急 |
| `scheduler.postTask` | 按声明的优先级排队 | 精细控制优先级（推荐） |
| `requestAnimationFrame` | 下次绘制前 | 动画、DOM 视觉更新 |
| `setTimeout(fn, 0)` | 下一个宏任务（最小约 4ms） | 粗暴让出主线程 |
| `requestIdleCallback` | 帧渲染完的空闲期 | 埋点、预取、非紧急计算 |

### 9. React 为什么没直接用 rIC

高频追问，答案很干脆：**rIC 触发时机太不可控（一忙就不跑）、优先级只有一档、而且 Safari 不支持**。所以 React Fiber 自己实现了一套 Scheduler，用 `MessageChannel` 制造宏任务来切片让出主线程，并自定义了多档优先级（lane 模型），这样跨浏览器行为一致、也能表达"这个更新比那个急"。

React 18+ 的 `useTransition` / `useDeferredValue` 就是这套调度能力对外的入口——你要的"低优先级更新"，用它比手撸 rIC 靠谱得多。

## 其实你每天都在用

- 所有滚动视差、拖拽、canvas 游戏循环，底层都是 rAF。
- Three.js / ECharts 的渲染循环就是 `requestAnimationFrame(render)`。
- 页面首屏渲染完在空闲时预取下一页数据、预热路由 chunk，是 rIC / `postTask('background')` 的经典场景。
- 埋点 SDK 把上报推迟到空闲期，就是为了不抢首屏和交互的时间。
- 长列表首次渲染分片（一次渲染 50 条，剩下的分批塞），用的就是"空闲期 + 分片"这套思路。
- Lighthouse 里那些标红的 Long Task，标准修法就是插 `await scheduler.yield()`。

## 常见误解（FAQ）

**❌ 误区1："rAF 就是 16.7ms 的定时器。"**
不是。它跟屏幕刷新同步，120Hz 屏上约 8.3ms 一次，低刷或页面卡顿时又会变慢；标签页隐藏还会自动暂停。硬编码 16 一定出问题。

**❌ 误区2："rIC 是异步执行，所以里面写多久都没关系。"**
恰恰相反。rIC 占的就是这一帧剩下的时间，超时就直接把下一帧的绘制和输入响应挤掉。必须靠 `timeRemaining()` 分片。

**❌ 误区3："rIC 一定会执行。"**
不加 `timeout` 的话，页面一直忙就可能永远不执行。要保证执行必须给 `{ timeout: ms }`。

**❌ 误区4："rIC 里改 DOM 挺方便的。"**
它跑在布局绘制之后，改 DOM 等于强制浏览器额外重排一遍。正确姿势是 rIC 算数据、rAF 改 DOM。

**❌ 误区5："scheduler.postTask 能把任务放到别的线程。"**
不能。它还是主线程，只是排队优先级不同。要并行只能 Web Worker。

**❌ 误区6："拆长任务用 `await new Promise(r => setTimeout(r, 0))` 就行了。"**
能用但不够好——`setTimeout` 让出后你的续作排在队尾，可能被后来的任务插到前面。`scheduler.yield()` 的续作是优先恢复的，这是它存在的意义。

**❌ 误区7："rIC 在 Worker 里也能用。"**
不能，Background Tasks API 不暴露给 Worker。Worker 里想分片就自己用循环 + `setTimeout`。

## 一句话总结

**rAF 在绘制前、跟刷新率同步，动画和 DOM 视觉更新只能用它；rIC 在绘制后的空闲期、必须看 `timeRemaining()` 分片且不能碰 DOM，还不是 Baseline；2026 年优化 INP 的正解是 `scheduler.postTask` 分优先级 + `scheduler.yield()` 拆长任务，配 `setTimeout` 兜底 Safari。**
