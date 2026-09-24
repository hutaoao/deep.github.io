---
layout: post
title: "WebView 内核原理与 JSBridge 通信机制"
date: 2026-12-19 00:00:00 +0800
categories: ["网络与性能", "浏览器原理"]
tags: [WebView, JSBridge, evaluateJavascript, URL Scheme, WKScriptMessageHandler, Hybrid]
description: >
  WebView 里到底跑着什么、JS 和原生为什么不能直接互调、三条桥通道各自"为什么能工作、什么情况下
  会静默失效"，以及回调 ID / 队列 / ready 时序这些工程细节的机制来源。
---

## 一句话概括

WebView 不是"应用里开个网页"，而是把一个浏览器内核（Android 上是 Chromium、iOS 上是 WebKit）嵌进 App 跑。于是有个绕不开的问题：**两套互不认识的运行环境，怎么互相调用？**

JSBridge 就是在回答这个问题，方向决定难度：

- **原生 → JS：容易。** WebView 本来就控制着 JS 引擎，往里注入一段代码执行就行。
- **JS → 原生：难。** JS 跑在沙箱里（在 Chromium 架构下连进程都是独立的），碰不到系统 API，所以必须借 WebView 提供的某个"钩子"把消息递出去。

这篇讲的是**机制**：三条通道为什么能工作、各自在什么情况下会**静默失效**（不报错、但消息就是收不到），以及回调 ID、队列、ready 时序这些工程手段到底在解决什么问题。至于 Android 侧的配置清单和安全加固，是另一套话题，这里只在必要处点一句。

先把结论摆出来，后面每条都会展开：

> **JSBridge 不是某个库给的能力，而是 WebView 暴露的几个钩子拼出来的约定。** 库做的事情只有三件——给调用编号（回调 ID）、把消息排队、加超时。你把这三点想明白，桥的原理就通了。

## 核心知识点

### 1. WebView 里到底跑着什么

先明确"物理位置"，后面所有限制都是它的推论。

| | Android WebView | iOS WKWebView |
| --- | --- | --- |
| 内核 | Chromium（Blink + V8） | WebKit（JavaScriptCore） |
| 更新方式 | 随系统或 Google Play 更新 | 随 iOS 版本更新 |
| 渲染进程 | 独立沙箱进程（宿主与渲染走 IPC） | WebContent 进程 |
| 崩溃表现 | 渲染进程崩了宿主还在，但页面变白 | 进程终止回调 |

三条关键推论：

**推论一：跨边界调用天生有成本。** JS 在渲染进程/渲染线程，原生逻辑在宿主线程，中间隔着 IPC 和两套 VM 的类型转换。所以**不要用桥传大对象、不要在高频路径（滚动、输入）上反复过桥**。官方也专门提醒过：传多兆字节字符串或二进制时得自己管内存，否则 32 位设备上容易 ANR 或崩溃。

**推论二：类型能过边界的很少。** Chromium 的 Java Bridge 文档列了允许通过的类型：**基本类型、一维数组、"类数组"的 JS 对象（有 `length` 的、以及 ES6 的 TypedArray）、以及已经注入过的 Java 对象**。没有嵌套对象、没有函数。所以桥的协议设计上，参数和返回值**一律走字符串（通常是 JSON）** 才是稳的。

**推论三：进程隔离换来的是稳定性，代价是宿主要主动善后。** 渲染进程可能因为内存被系统回收——Android 上是 `onRenderProcessGone`，宿主必须在这个回调里**销毁旧 WebView 并重建**，否则会拿到一个不可用的 WebView 实例持续报错。这也是"WebView 用久了整个页面变白、什么都点不动"的常见成因。

### 2. 原生 → JS：为什么这条路简单

因为 WebView 握着 JS 引擎，直接往里塞代码就行。

```kotlin
// Android：API 19+ 推荐。异步、能拿返回值
webView.evaluateJavascript("window.onUserInfo('$json')") { result ->
    // result 是 JS 侧表达式的返回值，字符串形式
}

// ❌ 老写法：没有返回值、失败无感知、还会混进导航机制里
webView.loadUrl("javascript:window.onUserInfo('$json')")
```

```swift
// iOS：也是异步 + 回调，页面没加载完调用会失败
webView.evaluateJavaScript("window.onUserInfo(\(jsonString))") { result, error in }
```

三个必须知道的点：

- **必须在页面加载完成后调用。** Android 等 `onPageFinished`，iOS 等 `didFinishNavigation`。提前调用不会抛明确异常，而是**静默什么都不发生**——这是"原生调 JS 没反应"的头号原因。
- **返回值是字符串化的。** `evaluateJavascript` 拿到的 `result` 是 JS 表达式结果的字符串表示；返回对象时通常只能拿到描述。需要结构化数据就让 JS 侧先 `JSON.stringify`，原生侧再解析。
- **字符串拼接是有风险的。** 上面那种 `"onUserInfo('$json')"` 拼法，只要参数里出现引号就会把脚本拼坏（本质是注入）。参数一律先编码（`JSON.stringify` + 转义），或者改用不拼字符串的通道。

### 3. JS → 原生：三条通道，为什么各自能工作

这是整道题的核心。三条通道的共性只有一句：**JS 只能通过"触发 WebView 本来就有的某种行为"，让原生在钩子里把这个行为截下来。** 三条通道的区别，就是"触发的是哪种既有行为"。

#### 通道一：URL Scheme 拦截（借"导航"当信道）

机制：JS 改一次地址（`location.href = 'myapp://...'`），WebView 视为一次**导航**，导航钩子就能拿到这个 URL；原生解析后返回"我自己处理"，真正加载被取消。

```kotlin
webView.webViewClient = object : WebViewClient() {
    override fun shouldOverrideUrlLoading(view: WebView, req: WebResourceRequest): Boolean {
        val uri = req.url
        if (uri.scheme == "myapp") {
            handle(uri.host, uri.query)   // 解析并执行
            return true                   // true = 自己消化，不交给 WebView 加载
        }
        return super.shouldOverrideUrlLoading(view, req)   // ✅ 其余交给默认实现
    }
}
```

它的**三个致命限制**都来自"这是导航"这个前提：

**限制一：连续调用会丢消息。** 这段 JavaScript 里第二条消息会被系统直接丢掉：

```js
location.href = 'myapp://log?data=111'   // 收到
location.href = 'myapp://log?data=222'   // 收不到，被导航机制过滤掉
```

原因很直白：**连续触发多次跳转时，WebView 会过滤掉后面的跳转请求。** 工程上有两种绕法——用**隐藏 iframe**（`iframe.src` 变化触发导航，但不像 `location.href` 那样互相吞掉），或者 JS 侧封一层**队列**、定期批量发送，保证极短时间内不会连发两次。

**限制二：参数受 URL 长度限制。** 传大参数会被截断，而且截断是静默的。所以参数必须有大小上限，超限改走"先落临时文件/存储再传引用"。

**限制三：`return true` 会连带拦掉自己的桥。** 有些桥的实现是"JS 把消息塞进队列 → 设 iframe 的 src 为某个特殊 scheme → 原生收到后取出队列"。如果你在 `shouldOverrideUrlLoading` 里对**所有** URL 都 `return true`，这条"通知原生来取队列"的信号同样会被吃掉，桥直接不工作。**规则是：只对自己认识的 scheme 返回 `true`，其余一律交回默认实现。**

优点也明确：**兼容性最好**（不依赖任何注入能力，老内核都能用），且**不暴露原生方法**，安全边界干净。

#### 通道二：弹窗拦截（借 `prompt` 的"返回值"当信道）

机制：JS 调 `prompt(一串 JSON)`，原生在弹窗钩子里拿到这段字符串，并且**可以给它一个返回值**。

```kotlin
webView.webChromeClient = object : WebChromeClient() {
    override fun onJsPrompt(
        view: WebView, url: String, message: String,
        defaultValue: String, result: JsPromptResult
    ): Boolean {
        return if (message.startsWith("jsbridge://")) {
            result.confirm(handle(message))   // ✅ 把结果同步回传给 JS
            true
        } else {
            super.onJsPrompt(view, url, message, defaultValue, result)  // ✅ 页面上真实的 prompt
        }
    }
}
```

它相对通道一有两个实打实的好处：

- **不发导航**，所以没有"连续调用丢消息"的问题。
- **能把结果同步返回给 JS**，而且**不需要暴露任何原生对象**。这是它相对通道三的关键差异——通道三也能同步拿返回，但代价是把原生方法摆在了页面能碰到的地方。

代价：得重写 `WebChromeClient`（并且要按前缀把页面上真实的 `prompt()` 交回默认实现，否则会破坏页面自己的逻辑），而且**iOS 的 UIWebView 不支持这个钩子**（WKWebView 支持，因为 WKWebView 把弹窗也做成了可拦截的 UI 代理）。所以它常被当成"兜底补位方案"。

#### 通道三：注入对象（现代主流）

机制：原生把**自己的一个对象**挂到 JS 的全局作用域上，JS 像调普通方法一样调它。

```kotlin
// Android：注入到 JS 的 window.Native
webView.addJavascriptInterface(AppBridge(), "Native")
```

```javascript
// JS 侧直接调，不需要初始化
Native.showToast('hello')
```

```swift
// iOS：没有"任意对象注入"，只有固定形态的一条消息通道
config.userContentController.add(self, name: "bridge")
```

```javascript
// iOS：对象固定挂在 window.webkit.messageHandlers 下
window.webkit.messageHandlers.bridge.postMessage({ action: 'share', text: 'hi' })
```

**这里有个高频考点：iOS 和 Android 的注入形态不一样。** Android 能注入任意对象、任意方法名；iOS 只能往 `window.webkit.messageHandlers.<name>` 这个固定位置注册**消息处理器**，JS 侧永远只有 `postMessage` 一个动作。所以 iOS 这条链是**单向的**（JS → Native），原生要回话得走 `evaluateJavaScript`。

再往下挖，有三个"官方文档才写、博客基本不写"的机制细节：

**细节一：`addJavascriptInterface` 对 JS 是同步阻塞的。** 官方对三种桥机制（`addWebMessageListener` / `postWebMessage` / `addJavascriptInterface`）的对照里，只有它是**同步**，并且特意解释了同步的含义：**JavaScript 执行环境会一直阻塞到原生方法返回为止**。也就是说 JS 侧调 `Native.foo()` 是能直接拿到返回值的——这是它的优点；但反过来说，原生方法里只要有耗时操作（读文件、网络、查数据库），页面的 JS 线程就被卡住了。**优点和陷阱是同一件事。**

**细节二：原生方法跑在 WebView 的私有后台线程上。** 官方文档和 Chromium 的实现说明都写了这点：JS 发起的这次交互，整个要在后台线程上处理完。推论很实际——**不能在里面直接改 UI**，要切回主线程：

```kotlin
@JavascriptInterface
fun showToast(msg: String) {
    // ❌ 直接操作 UI 会抛 CalledFromWrongThreadException
    // activity.binding.text.text = msg

    // ✅ 切回 UI 线程
    activity.runOnUiThread { activity.binding.text.text = msg }
}
```

顺带一个很坑的连带效果：**后台线程里未捕获的异常，会被 WebView 当成 JS 错误处理**，你只能看到一句 `Java exception was raised during method invocation`，既没有原因也没有行号。所以桥的方法体里必须自己 try/catch 兜住。

**细节三：注入对象的增删，要等下一次页面加载才在 JS 侧生效。** Chromium 的 Java Bridge 实现说明里明确写了这一点：`addJavascriptInterface` / `removeJavascriptInterface` 的增删**不会在 JavaScript 侧立刻反映出来，要等下一次页面加载**。所以工程上的铁律是：**`addJavascriptInterface` 必须在 `loadUrl` 之前调用**；页面已经加载完了再注入，当前页面根本调不到，得等下次加载。

最后是安全。官方安全文档的措辞很硬：

- `addJavascriptInterface` 会把对象注入到 WebView 的**每一个 frame**（包括 iframe），而且**没有任何机制能校验调用方来自哪个帧**。
- **不要依赖 `WebView.getUrl()` 之类的返回值做安全校验**——WebView 的行为是异步的，这个方法无法保证准确，也不会告诉你到底是哪个 frame 发起的请求。
- API 17 之前的版本会暴露对象上**所有 public 方法**（连 `Object` 继承来的也算），可以用反射链执行任意代码（CVE-2012-6636）；API 17 起只有加了 `@JavascriptInterface` 的方法才暴露。
- 注解只是**缩小暴露面**，不是"安全了"。注入的方法以 App 的权限执行，参数来自网页——**一律当不可信输入**，白名单校验。

> 官方现在推荐的形态是 `WebViewCompat.addWebMessageListener`：带 `sourceOrigin`，可以做来源白名单校验，双向、异步。它是"通道三"的现代版本，我们这里只当作演进方向记一句。

### 4. 三条通道怎么选

把三个机制的差异放一张表里，面试答到这里基本就满分了：

| | URL Scheme 拦截 | `prompt` 拦截 | 注入对象 |
| --- | --- | --- | --- |
| 借用的既有行为 | 导航 | 弹窗 | JS 全局对象 |
| 原生官方推荐度 | — | — | 旧机制，不推荐 |
| 连续调用 | **会丢消息**，要 iframe 或队列 | 不丢 | 不丢 |
| 参数长度 | 受 URL 长度限制 | 不受限 | 不受限 |
| 单次调用能否同步拿结果 | **不能**（发出去就没有回程） | 能 | 能（JS 同步阻塞到方法返回） |
| 性能 | 差（走导航链路） | 中 | 好 |
| 兼容性 | 最好 | 好（UIWebView 不支持） | 好 |
| 安全边界 | 好（不暴露方法） | 中 | **差**（暴露即攻击面） |
| 适合场景 | 老内核兜底、跳转类外链 | 需要同步返回、通道一的补位 | 完全可信的 H5 |

**实操结论**：可控的 H5（自己打包或白名单域名）用注入对象或 `addWebMessageListener`；需要跨信任边界、或者要兼容老内核，就退回 URL Scheme；`prompt` 的价值在于"能同步返回，又不用把任何原生对象暴露给页面"，代价是会被页面上真实的 `prompt()` 弹窗共用同一个钩子。

### 5. ready 时序：为什么桥必须有"待发送队列"

这是桥最容易出线上事故的地方，而且跟通道选型无关。

两套时序是完全独立的：**H5 的脚本在页面一加载就会跑，而桥对象是原生在某个时刻注入/注册的。** 谁先谁后没有保证。于是页面刚出来就调桥，很可能命中"桥还没就绪"，表现就是**点按钮没反应、过一会儿又好了**。

标准做法是一个 `ready` 标记 + 一个待发送队列：

```js
const JSBridge = {
  ready: false,
  queue: [],

  call(method, params) {
    if (!this.ready) {
      this.queue.push([method, params])   // ✅ 没就绪先入队，不丢
      return
    }
    this._send(method, params)
  },

  onReady() {
    this.ready = true
    this.queue.forEach(([m, p]) => this._send(m, p))   // ✅ 就绪后补发
    this.queue.length = 0
  },

  _send(method, params) {
    prompt(`jsbridge://${method}?params=${encodeURIComponent(JSON.stringify(params))}`)
  },
}
```

原生侧配合的两点：

- **Android 要在 `onPageStarted` 阶段就注入**（页面还没开始跑脚本）。等 `onPageFinished` 再注入，首屏那批 JS 早就执行完了——它们看到的是一个没有桥的世界。
- **iOS 用 `WKUserScript` 时要选对 `injectionTime`。** 想让"页面自己的脚本执行之前桥就装好"，得用 `.atDocumentStart`；`.atDocumentEnd` 是文档加载完之后才注入，首屏同步脚本里调桥会扑空。

### 6. 一个能背下来的最小桥协议

把上面所有机制收进一段代码，面试被问"如果让你设计一个 JSBridge，你怎么做"就能直接讲：

```js
let seq = 0
const cbs = new Map()          // 回调 ID → 待兑现的 Promise

function invokeNative(method, params, timeout = 10000) {
  const id = `cb_${++seq}`
  return new Promise((resolve, reject) => {
    cbs.set(id, { resolve, reject })

    // 1）带上回调 ID 和参数（参数必须编码，走 URL 通道尤其必须）
    prompt(`jsbridge://${method}?id=${id}&params=${encodeURIComponent(JSON.stringify(params))}`)

    // 2）必须有超时：桥可能是"哑失败"，Promise 会永远挂着
    setTimeout(() => {
      if (cbs.delete(id)) reject(new Error(`bridge timeout: ${method}`))
    }, timeout)
  })
}

// 3）原生执行完，通过 evaluateJavascript 回调这个函数
function onNativeResult(id, json) {
  const cb = cbs.get(id)
  if (!cb) return                 // 已超时或被别的路径处理过
  cbs.delete(id)
  cb.resolve(JSON.parse(json))
}
```

三件事对应三个痛点：

- **回调 ID** 让桥支持并发——原生执行完靠 ID 找回对应的 Promise，而不是靠"谁先谁后"。
- **超时** 是唯一的兜底。桥的失败模式几乎都是静默的：JS 没发出去、原生没解析出来、原生处理时抛了异常、宿主线程被占住。超时是唯一能把这些情况变成"可见错误"的手段。
- **参数编码** 因为原生侧只应该把传来的东西当字符串处理，绝不能当可信结构。

## 其实你每天都在用

- 在微信里点 H5 的"分享给朋友"，弹出来的原生分享面板就是桥调起来的。
- 网页里点"扫一扫"能打开相机，也是桥；H5 自己没这个权限。
- 点"去支付宝付款"时浏览器弹一句"即将离开当前页面"，那是一次 URL Scheme 导航被浏览器拦下了。
- 上传头像时点"选择图片"，弹出来的原生相册是 WebView 的文件选择钩子。
- 有的 App 里网页 `alert` 出来的是 App 自己画的漂亮弹窗，因为原生把弹窗钩子接管了。
- 网页顶部横幅写"在 App 内打开体验更佳"，判断的是 User-Agent，跳转靠的也是 scheme。
- H5 页面里"下载文件"经常失败，因为下载、写文件、权限都得原生化去做。
- 在 H5 里退出登录后切账号仍然显示上一个账号，是 Cookie 和 WebView 存储没清。
- H5 首屏刚出现时点按钮没反应、过一两秒就好了——桥的 ready 竞态。
- 页面用久了整片变白、点哪都没反应——渲染进程被系统回收了，宿主没在崩溃回调里重建 WebView。

## 常见误解（FAQ）

**❌ 误区1："JSBridge 是第三方库提供的能力。"**
桥是 WebView 自己暴露的钩子（导航拦截 / 弹窗拦截 / 对象注入）拼出来的约定。库只做三件事：给调用编回调 ID、待发送队列、超时。所以"不用库能不能做桥"这个追问，答案是能——只是要自己处理这三个问题。

**❌ 误区2："JS 调原生是同步的，可以直接拿返回值。"**
要看通道。URL Scheme 通道**永远拿不到同步返回值**（它是发导航，没有回程），所以整条桥的协议必须设计成异步（回调 ID + 回调表 + 超时）。`prompt` 和 `addJavascriptInterface` 确实能同步返回，但它们各自的代价分别是"共用真实弹窗钩子"和"暴露原生方法 + 卡住 JS 线程"。**"能同步"不等于"应该同步用"**——工程上稳定做法仍然是统一异步协议。

**❌ 误区3："`location.href` 连着调两次，原生能收到两条。"**
收不到，**第二条会被直接丢掉**——连续触发导航时 WebView 会过滤掉后面的跳转请求。要连发就得用隐藏 iframe，或者 JS 侧排队节流。

**❌ 误区4："在 `shouldOverrideUrlLoading` 里对所有 URL 都 `return true` 最省事。"**
会把桥自己的信号一起拦掉。有些桥实现是"JS 入队 → iframe 触发特殊 scheme → 原生收到后取队列"，你无脑 `true` 就等于把"来取队列"的通知吃了。只对自己认识的 scheme 返回 `true`，其余交回默认实现。

**❌ 误区5："`addJavascriptInterface` 的方法在 UI 线程执行，可以直接改控件。"**
官方文档明确写的是跑在 WebView 的**私有后台线程**上。直接改 UI 会抛 `CalledFromWrongThreadException`；而且后台线程里未捕获的异常会被当成 JS 错误上报，你只能看到一句 "Java exception was raised during method invocation"，没有堆栈信息。桥的方法体要自己 try/catch，改 UI 要切主线程。

**❌ 误区6："调用 `addJavascriptInterface` 之后，当前页面立刻就能用 `Native.xxx()`。"**
不行。Chromium 的 Java Bridge 实现说明写了：注入对象的**增删要等下一次页面加载才在 JS 侧反映出来**。所以必须在 `loadUrl` 之前注入；页面加载完再注入，当前页面调不到。

**❌ 误区7："iOS 也能像 Android 一样往 window 上注入任意对象。"**
不能。iOS 只有 `window.webkit.messageHandlers.<name>.postMessage()` 这条固定形态的通道，而且**是单向的**（JS → Native），原生回话必须走 `evaluateJavaScript`。

**❌ 误区8："加了 `@JavascriptInterface` 注解就安全了。"**
注解解决的是 API 17 之前"所有 public 方法（含反射链）都能被调用"那个漏洞，它只**缩小暴露面**。官方安全文档的原文口径是：对象会注入到**所有 frame（含 iframe）**，且**没有机制校验调用来源**；也不能靠 `WebView.getUrl()` 做来源校验（WebView 行为异步，结果不可信）。所以注解之后依然要：不对不可信内容开桥、参数白名单校验、方法体自己兜异常。

**❌ 误区9："`evaluateJavascript` 在页面加载完成前也能执行。"**
执行不了，而且不会抛出明显错误，是**静默失败**。必须等 `onPageFinished`（iOS 等 `didFinishNavigation`）。

**❌ 误区10："WebView 和手机浏览器内核一样，行为就一样。"**
内核同源，但**版本和裁剪都不同**：Android WebView 随系统/Play 更新，WKWebView 随 iOS 更新，能力集和 bug 集都不一样。这也是为什么 WebView 里的问题必须真机验证，模拟器和桌面浏览器都可能骗你。

## 一句话总结

**WebView 是把浏览器内核嵌进 App、渲染跑在独立进程里的容器，这决定了它和原生之间只能少量、按协议过桥（工程上稳定做法是一律异步）；原生调 JS 直接用 `evaluateJavascript`，JS 调原生只有三条借来的通道——URL Scheme（会丢消息、受长度限制、拿不到同步返回）、`prompt`（能同步返回但会占用真实弹窗钩子）、注入对象（性能最好还能同步返回，但暴露即攻击面，且 iOS 只有单向的 `messageHandlers`）；桥的工程核心只有三件事：回调 ID 支持并发、队列解决 ready 时序、超时兜住静默失败。**
