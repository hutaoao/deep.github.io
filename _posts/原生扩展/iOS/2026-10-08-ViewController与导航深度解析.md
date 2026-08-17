---
layout: post
title: "ViewController 与导航：iOS 页面管理机制深度解析"
date: 2026-10-08
categories: ["原生扩展", "iOS"]
tags: [ViewController, 生命周期, UINavigationController, 模态展示, NavigationStack, 容器控制器]
description: >
  面试高频：UIViewController 生命周期每个方法该干什么、UINavigationController 栈怎么玩、模态与 SwiftUI NavigationStack 的区别，一篇讲透 iOS 页面管理。
---

## 一句话概括

iOS 里"一个页面 = 一个 `UIViewController` 实例"，它不像 Flutter 的 Widget 那样用完即弃，而是有完整的**生命周期**：从创建、视图加载、出现/消失到最终释放，每个时机都有该干的事。而 UINavigationController（导航栈）、模态展示、容器控制器（Tab/Page）构成了一套**命令式**的页面路由体系——你直接 push、pop、present，而不是声明"页面之间的关系"。

这套体系和 Flutter 的 `Navigator`、RN 的 `React Navigation` 在理念上相通，但 iOS 更"实体化"、更显式：VC 是活的实例，栈是真实的数组，生命周期贯穿开发始终。**跨端同学面试最容易被追问的就是"viewDidLoad 和 viewWillAppear 到底有什么区别""为什么 pop 回来状态没重置""SwiftUI 的 NavigationStack 和 UIKit push 是一回事吗"**——这些正是本文要敲死的点。

## 核心知识点

### 1. UIViewController 生命周期：六个时机各司其职

生命周期可以记成一句话：**"建 → 载 → 现 → 隐 → 销"**。完整顺序（首次进入）：

```swift
// 首次加载：init → loadView → viewDidLoad → viewWillAppear → viewDidAppear
// 每次再次出现：            → viewWillAppear → viewDidAppear
// 离开当前页：                              → viewWillDisappear → viewDidDisappear
// 真正销毁：                                → viewWillDisappear → viewDidDisappear → deinit
```

| 方法 | 调用次数 | 该干什么 | 不该干什么 |
|---|---|---|---|
| `viewDidLoad` | **整个生命周期只一次** | 初始化 UI、绑定数据模型、注册通知、发首屏请求 | 放"每次出现都要刷新"的逻辑 |
| `viewWillAppear` | 每次出现都调 | 刷新数据、更新导航栏/状态栏、注册键盘监听 | 做重活（用户还没看到页面） |
| `viewDidAppear` | 每次出现后 | 开启动画、播视频、埋点、传感器监听 | 放初始化（太晚了） |
| `viewWillDisappear` | 每次离开前 | 收起键盘、保存草稿、停止动画、注销监听 | 做"最终销毁"（页面可能只是被盖住） |
| `viewDidDisappear` | 每次离开后 | 暂停视频、停资源密集操作 | 以为 VC 一定被释放了（不一定） |
| `deinit` | 仅一次 | 取消网络请求、释放大资源 | 忘记打破循环引用导致根本不进这里 |

最容易踩的坑：`viewDidLoad` 只调一次，所以"从详情返回列表再进详情"时，`viewDidLoad` **不会**再跑——如果你把刷新逻辑写在 `viewDidLoad`，就会看到旧数据。正确的做法：**一次性初始化放 `viewDidLoad`，每次刷新放 `viewWillAppear`**。

`loadView()` 一般不需要自己重写——用 Storyboard 系统自动加载，纯代码就自己建 `view` 并赋值 `self.view`。`deinit` 里的循环引用问题见 FAQ。

### 2. UINavigationController：命令式的导航栈

导航栈本质就是个 `UIViewController` 数组，栈底是根 VC，栈顶是当前页。核心就是 push / pop：

```swift
// push：新页面从右侧滑入，自动带返回按钮
let detail = DetailViewController()
detail.itemID = selectedID           // 顺手把数据传过去
navigationController?.pushViewController(detail, animated: true)

// pop：回到上一页 / 根页 / 指定页
navigationController?.popViewController(animated: true)
navigationController?.popToRootViewController(animated: true)
navigationController?.popToViewController(targetVC, animated: true)
```

几个面试常问的属性：

```swift
let stack = navigationController?.viewControllers      // 整个栈（数组）
let root  = navigationController?.viewControllers.first // 栈底根 VC
let top   = navigationController?.topViewController     // 栈顶 VC
let visible = navigationController?.visibleViewController // 当前"看得见"的 VC
```

**`topViewController` vs `visibleViewController`** 的区别：`top` 是栈顶那个 VC；`visible` 是当前屏幕上真正显示的——当你再 present 一个模态页盖在上面时，`visibleViewController` 变成那个模态页，而 `topViewController` 还是导航栈顶。调试"我现在到底在哪个页面"时优先看 `visibleViewController`。

配置导航栏用 VC 自己的 `navigationItem`，不是直接改 `navigationBar`：

```swift
title = "详情"
navigationItem.rightBarButtonItem = UIBarButtonItem(
    systemItem: .add, target: self, action: #selector(onAdd)
)
// 隐藏返回按钮文字（只留箭头）
navigationItem.backButtonTitle = ""
```

> 跨端对照：Flutter `Navigator.push/pop` 概念一致，但 Flutter 1.0 是命令式（拿 `BuildContext` 找 Navigator），**Navigator 2.0** 才变成声明式的 `Router`；RN 的 `React Navigation` 从一开始就是声明式。iOS 在 **UIKit 层至今都是命令式**，声明式是 SwiftUI 才有的（见第 4 节）。

### 3. 模态展示：独立于栈的"盖楼"

模态（present / dismiss）不走导航栈，适合"必须完成才能走"的任务（登录、设置、分享）。iOS 13 起**默认样式从 `.fullScreen` 变成了 `.pageSheet`（卡片式）**，这个改动坑了不少人：

```swift
let login = LoginViewController()
login.modalPresentationStyle = .fullScreen   // ✅ 全屏覆盖，下面页面会走 viewWill/DidDisappear
// login.modalPresentationStyle = .pageSheet // ❌ 默认卡片式：下面页面"还露着"，且不走 disappear

present(login, animated: true) { print("展示完成") }
dismiss(animated: true)                       // 由被 present 的 VC 自己调用
```

常见 `modalPresentationStyle`：

- `.fullScreen`：全屏，覆盖页生命周期正常触发
- `.pageSheet`（默认）：卡片，下滑可关闭，覆盖页不触发 disappear
- `.formSheet`：iPad 上的小卡片
- `.overFullScreen` / `.overCurrentContext`：透明背景，下层可见但不可交互（做半透明弹层用）

```swift
// ❌ 卡片样式下用户一滑就关了，关键流程被打断
login.modalPresentationStyle = .pageSheet

// ✅ 必须完成的流程，禁用下滑关闭
login.isModalInPresentation = true   // 配合 .pageSheet 使用，禁止手势 dismiss
```

模态页里自己调 `dismiss(animated:)` 即可关闭；如果模态里还要多级跳转，把模态页包一层 `UINavigationController` 再 present：

```swift
let nav = UINavigationController(rootViewController: login)
nav.modalPresentationStyle = .fullScreen
present(nav, animated: true)
```

### 4. SwiftUI NavigationStack：声明式的新选择（iOS 16+）

UIKit 是命令式，而 SwiftUI 从 iOS 16 起提供了声明式的 `NavigationStack`，更接近 Flutter/RN 的思维：

```swift
// ✅ SwiftUI 声明式导航：描述"去哪"，不是"怎么推"
NavigationStack {
    List(items) { item in
        NavigationLink(item.title, value: item.id)   // 用 value 关联数据
    }
    .navigationDestination(for: Item.ID.self) { id in
        DetailView(itemID: id)                        // 目标页由类型自动决定
    }
}
```

用 `NavigationPath` 还能做**编程式跳转/返回**（相当于 pop 到任意位置）：

```swift
@State private var path = NavigationPath()
NavigationStack(path: $path) {
    // ...
}
// 跳转：path.append(itemID)
// 返回根：path.removeLast(path.count)
```

对比一句话：**UIKit 的 push 是你手动把 VC 塞进栈数组；SwiftUI 的 NavigationStack 是你声明"页面图"，框架帮你维护栈**。底层最终还是那套 `UINavigationController`，只是封装层变厚了。

### 5. 容器控制器：页面里的页面

`UINavigationController`、`UITabBarController`、`UIPageViewController` 都是"容器控制器"——它们管理一组子 VC，并负责调用子 VC 的生命周期。你也可以自造容器：

```swift
// 把 child 加进容器（顺序别乱）
addChild(child)
child.view.frame = containerView.bounds
containerView.addSubview(child.view)
child.didMove(toParent: self)

// 移除
child.willMove(toParent: nil)
child.view.removeFromSuperview()
child.removeFromParent()
```

`UITabBarController` 是每个 Tab 各带一个导航栈，切 Tab 时**默认保留各 Tab 的状态**（不像很多 RN 实现切走就重绘）；`UIPageViewController` 做左右翻页（引导页、电子书）。

> 关键区分：**容器控制器 vs 直接 addSubview 一个 UIView**。前者子 VC 的生命周期（旋转、内存警告、appear/disappear）会被正确转发；后者只是视图嵌套，没有任何生命周期回调。承载独立业务逻辑的，用容器；纯 UI 组件，用 UIView。

## 其实你每天都在用

- **列表点 cell 进详情**：`didSelectRowAt` 里 `pushViewController(detail)`，顺手把 `itemID` 传过去
- **下拉刷新后回列表看新数据**：其实是在 `viewWillAppear` 里重新拉接口，而不是依赖 `viewDidLoad`
- **点击"收藏"未登录弹登录框**：用 `.fullScreen` 模态 present，登录成功 `dismiss` 后刷新原页
- **底部 Tab 切换**：`UITabBarController` 切走再切回，原来页面的滚动位置还在（状态被保留）
- **左滑返回上一级**：`navigationController?.interactivePopGestureRecognizer` 提供的系统侧滑手势
- **SwiftUI 里 `NavigationLink` 跳转**：底层就是 `NavigationStack` 在管理栈
- **引导页左右滑**：`UIPageViewController` 的 `.scroll` 转场
- **自定义 push 转场动画**：重写 `UIViewControllerAnimatedTransitioning` 挂到 `navigationController?.delegate`

## 常见误解（FAQ）

**❌ 误区一："viewDidLoad 和 viewWillAppear 都能加载数据，放哪都行"**

这是面试最高频失分点。`viewDidLoad` **整个生命周期只调用一次**（视图首次从 nib/代码加载完），`viewWillAppear` **每次页面要出现都调用**（包括从下级返回）。把"每次都要最新的数据"（如消息列表）放 `viewDidLoad`，就会看到返回后数据不刷新。经验法则：**初始化性配置放 `viewDidLoad`，刷新性传播放 `viewWillAppear`**。

**❌ 误区二："pop 回来状态没重置，是因为 ViewController 被系统回收了"**

恰恰相反——pop 之后 VC **没被释放**（多半是循环引用卡住了，比如 delegate 没用 `weak`、`closure` 捕获了 `self`），所以它还保持着上次的状态，再 push 进来时 `viewDidLoad` 不会再跑。解决：① 用 `weak` 打破循环引用；② 在 `viewWillAppear` 里主动重置 UI（清输入框、复位开关）；③ 需要全新实例就每次都 `push` 新建的 VC。

**❌ 误区三："present 一个模态页，下面的页面会走 viewDidDisappear"**

只有 `.fullScreen` / `.overFullScreen` 这类"完全盖住"的样式，下面页面才会走 `viewWill/DidDisappear`。**iOS 13 默认的 `.pageSheet` 卡片样式，下面页面依然露在后面，不会触发 disappear**，也不会被暂停。如果你的逻辑依赖"页面被盖住就停动画/注销监听"，要用 `.fullScreen` 或在 `present` 的 VC 里自己管理。

**❌ 误区四："SwiftUI 的 NavigationStack 和 UIKit 的 pushViewController 是一回事"**

能力上等价（都是维护一个导航栈），但**范式不同**：UIKit 是命令式——你自己 `push`/`pop` 一个 VC 实例；SwiftUI 是声明式——你描述"页面图 + 数据 value"，框架托管栈。更深的区别：SwiftUI 的页面是值类型 `View`（每次 body 重算），UIKit 的 VC 是引用类型实例（长期存活、有状态）。混用（SwiftUI 嵌入 UIKit 用 `UIHostingController`）时，生命周期归容器 VC 管，这点面试常被追问。

## 一句话总结

iOS 的页面管理就三句话：**ViewController 是活的页面实体，生命周期六个时机各司其职（`viewDidLoad` 初始化、`viewWillAppear` 刷新、`deinit` 收尾）；UINavigationController 是命令式导航栈（push/pop 直接操作），模态展示是独立于栈的"盖楼"，SwiftUI 的 NavigationStack 则把这套栈包成了声明式**——懂了"实体 + 显式栈 + 生命周期"这三件套，你在跨端项目里写原生扩展、调原生崩溃时，脑子里就不再是黑盒。
