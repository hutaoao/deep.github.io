---
layout: post
title: "RN vs Flutter 架构对比深度解析：面试官真正关心的5个问题"
description: >
  对比题：React Native（桥接/JSI + 原生组件）与 Flutter（自绘引擎）的架构差异。
  讲清两者渲染与通信模型，掌握后能答出「为什么 Flutter 更可控、RN 更贴近原生」。
date: 2026-05-31
categories: [跨端开发, 技术对比]
tags: [React Native, Flutter, 架构对比, 面试]
---

## 一句话概括

React Native 让 JS 驱动原生控件（"和原生做朋友"），Flutter 用 Skia 自绘所有像素（"做自己的世界"）——面试官不在乎你背出几层架构，只在乎你有没有理解 **跨语言通信开销** 和 **渲染路径差异** 对性能的根本影响。

## 核心知识点

### 1. RN 传统 Bridge：异步 JSON 队列是瓶颈

每次 JS ↔ Native 通信都序列化 JSON，高频交互（手势拖拽、实时动画）会被队列堵住。

```javascript
// 旧架构：JS 调原生 → JSON 序列化 → 丢进 Bridge 队列 → 等原生线程拿
NativeModules.Camera.takePhoto(); // 异步，不知道什么时候真正执行
// 100 次/秒的手势回调 → Bridge 队列爆炸 → 掉帧
```

### 2. RN 新架构（JSI + Fabric）：让 JS 直接拿 C++ 指针

JSI 的核心价值：JS 可以持有 C++ 对象引用并**同步调用**方法，无需序列化、不走队列。

```typescript
// 新架构 JSI：同步调用，零序列化
const result = NativeModules.Crypto.encrypt('hello', 'key'); // 直接返回，不是 Promise
// Fabric 渲染器：JS 和 Shadow Tree 之间是同步通信
```

面试关键句："JSI 让 JS 直接操作 C++ 对象，Fabric 渲染器把 Shadow Tree 搬到 C++ 层——本质上是在消灭 Bridge。"

### 3. Flutter 自绘引擎：Dart 层搞定一切 UI，Platform Channel 只调原生 API

Flutter 的所有控件（Button、Text、ListView）都是由 Skia/Impeller 直接画到 GPU 上的，不依赖 iOS UIKit 或 Android View。

```dart
// Dart → C++ 引擎是 FFI 直接调用，没有跨语言序列化
// Platform Channel 只在访问原生 API（相机、蓝牙）时使用
static const platform = MethodChannel('com.example/battery');
final level = await platform.invokeMethod<int>('getBatteryLevel');
// UI 更新、手势、动画 → 全在 Dart 层，零开销
```

### 4. 性能差异的根因：跨语言调用次数

| 场景 | RN（传统） | RN（新架构） | Flutter |
|------|-----------|------------|---------|
| 列表快速滚动 | 60fps 勉强 | 60fps 稳定 | 60/120fps 满帧 |
| 手势拖拽动画 | 45-55fps | 55-60fps | 满帧 |
| CPU 密集计算 | AOT + JS JIT | JSI 同步调用 | Dart AOT（原生机器码） |
| 包体积 | ~5-8 MB | ~5-8 MB | ~10-15 MB |

**一句讲透**：RN 每帧更新都要跨语言，Flutter 的所有渲染都在 Dart 层完成，这是性能差异的根源，不是语言或框架设计的问题。

### 5. 选型决策：不问"哪个好"，问"团队有什么"

```
团队会 React/TS → RN 三天上手
团队从零开始 → Flutter 学习路径更清晰
需要融入 iOS/Android 原生体验 → RN（渲染原生控件）
需要全平台像素级一致 → Flutter（自绘一切）
需要桌面/Web 扩展 → Flutter（Windows/macOS/Linux/Web）
包体积敏感（IoT）→ RN（体积更小）
```

## 其实你每天都在用

- **微信小程序的 WXS** 试图在渲染层做计算避免跨线程通信——就是在解决 RN Bridge 的同款问题
- **React 的 Concurrent Mode** 和 Flutter 的渲染管道一样，都是"可中断的渲染管线"思想
- **Expo** 封装 RN 新架构，让开发者不用手动配 JSI，这和 Flutter 的"自带引擎"是异曲同工的思路
- **WebAssembly** 让浏览器也能 AOT 编译——和 Dart 的 AOT 编译谋的是同一个"消灭 JIT 开销"的局
- **鸿蒙 ArkTS 的 NAPI** 干的事和 RN 的 JSI 完全一样——让 JS/TS 直接调用 C/C++

## 常见误解（FAQ）

**❌ 误区：Flutter 没有 bridge，所以比 RN 快。**

✅ Flutter 有 Platform Channel，只是它和 UI 无关。UI 层都在 Dart VM 内完成，不需要跨语言。RN 传统架构每帧更新都要走 Bridge——这是差异的根因。但 RN 新架构的 JSI 正在接近 Flutter 的零开销水平，两者的差距在缩小。

**❌ 误区：Dart 比 JavaScript 快，所以 Flutter 性能更好。**

✅ Dart AOT 编译成原生机器码确实比 JS JIT 快 2-3 倍在 CPU 密集场景，但 UI 性能差异主要来自渲染路径（跨语言次数），不是语言速度。你把 JS 换成 Dart 跑在 RN 架构里，一样掉帧。

**❌ 误区：RN 新架构出来后，Flutter 就没优势了。**

✅ JSI 解决了通信瓶颈，但渲染方式没变——RN 仍然渲染平台原生控件，Flutter 仍然自绘。这意味着 RN 新架构下你依然受限于原生控件的表现力（比如 iOS 的 UIScrollView 在极端复杂场景的性能天花板），而 Flutter 可以自己优化渲染管线。

**❌ 误区：Flutter 的 Impeller 引擎替换了 Skia 引擎。**

✅ Impeller 不是替换，是**重新实现**——它针对移动端做了一套 AOT 着色器方案，彻底消除了 Skia 的运行时着色器编译卡顿（shader jank）。目前的 Flutter 3.x 中 Impeller 是默认引擎，但 Skia 仍然是 Web 端的唯一选择。

## 一句话总结

RN 的路线是"把桥磨薄"（JSI→消灭序列化），Flutter 的路线是"把引擎做强"（Impeller→消灭着色器卡顿）——两条路都在往同一个终点跑：让跨端开发有纯原生的性能。

