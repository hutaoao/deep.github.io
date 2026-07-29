---
layout: post
title: "RN vs Flutter 通信机制对比：从 JSON Bridge 到 JSI 到 Platform Channel"
date: 2026-06-01
categories: [跨端开发, 技术对比]
tags: [React Native, Flutter, Bridge, JSI, Platform Channel, 面试]
---

## 一句话概括

跨端框架的性能瓶颈往往不在"谁的语言更快"，而在**JS/Dart 与原生之间的通信路径有多长**——RN 从 JSON Bridge 演进到 JSI，Flutter 用 Platform Channel + FFI 两套体系，背后都是在做同一件事：让跨语言调用更接近一次普通函数调用。

## 核心知识点

### 1. RN Bridge 的代价：每次调用都是一次 JSON 往返

```javascript
// 旧架构：JS 调原生 → JSON.stringify → 丢队列 → Native 轮询 → JSON.parse → 执行
await NativeModules.Calendar.createEvent('周会', '3楼');
// 这行代码背后：2 次序列化 + 2 次反序列化 + 线程切换
// 如果传 10MB 图片 Base64 → JSON 体积膨胀到 13.3MB
```

### 2. JSI 的革命：JS 直接拿着 C++ 对象的引用

新架构的核心不是优化了序列化，是**消灭了序列化**：

```cpp
// C++ 侧：TurboModule 通过 JSI 直接暴露函数给 JS
jsi::Value multiply(jsi::Runtime &rt, jsi::Value a, jsi::Value b) {
  return jsi::Value(a.asNumber() * b.asNumber());
  // JS 调用这个函数 → 直接进入 C++ 代码 → 零拷贝返回
}
```

面试加分：JSI 的 `jsi::Runtime` 是抽象层，可以对接 Hermes / JSC / V8 三种引擎，这是 RN 能同时跑在 iOS 和 Android 上的关键。

### 3. Flutter Platform Channel：二进制编码的异步通道

Flutter 不用 JSON，用的是 `StandardMethodCodec`——紧凑的二进制格式：

```dart
// Dart 端
final level = await platform.invokeMethod<int>('getBatteryLevel');
// 底层：Dart → 二进制编码 → Engine 转发 → Native 解码执行 → 二进制返回 → Dart 解码
```

对比 JSON：一个数字 `42` 在 JSON 里是 `"42"`（2 字节），在 StandardMethodCodec 里是 `0x03 0x2A`（2 字节）——效率差不多但省了 parse 开销。大数据（10MB 图片）用 BinaryCodec 比 JSON 快 2-3 倍。

### 4. Dart FFI：同步调用 C/C++ 的终极武器

Platform Channel 的"异步"是硬伤——高频场景（传感器每秒 100 次回调）不行。FFI 直接同步调用：

```dart
final lib = DynamicLibrary.open('libnative.so');
final int Function(int, int) add = lib
    .lookupFunction<Int32 Function(Int32, Int32), int Function(int, int)>('add');
print(add(3, 4)); // 同步返回 7，零切换开销
```

**一句讲透**：MethodChannel = 异步（跨线程），FFI = 同步（同线程）。选前者做业务 API 调用，选后者做高性能计算。

### 5. 性能实战对比：10MB 文件读取谁最快

| 方案 | 耗时 | 峰值内存 | 拷贝次数 |
|------|------|---------|---------|
| RN Bridge + Base64 | ~850ms | 28MB | 3次 |
| Flutter MethodChannel | ~320ms | 15MB | 1次 |
| RN JSI TypedArray | ~120ms | 11MB | **0次** |

JSI TypedArray 的"零拷贝"是关键——Dart FFI 也可以做到同样的效果（传指针而非拷贝数据）。

## 其实你每天都在用

- **微信小程序的 WXS** 在渲染线程执行脚本，避免逻辑层↔渲染层的通信延迟——和 RN 消灭 Bridge 是同类思路
- **Chrome 的 `postMessage`** 在 Web Worker 间传数据需要序列化（结构化克隆），和 RN Bridge 的 JSON 模型一样有拷贝开销
- **Node.js 的 C++ Addon (napi)** 和 Flutter 的 FFI 如出一辙——都是 JS/Dart 调用 C 函数的桥
- **Android 的 JNI** 是 RN JSI 的灵感来源——让上层语言直接操作 C++ 对象
- **鸿蒙 ArkTS 的 NAPI** 玩法和 JSI 完全一致：`napi_value` 类型映射 + 同步/异步双模式

## 常见误解（FAQ）

**❌ 误区：Flutter 没有 Bridge，所以通信比 RN 快。**

✅ Flutter 有 Platform Channel（异步二进制通道），只是它**不参与 UI 渲染**。UI 层全在 Dart VM 内完成才是快的真正原因。如果你频繁通过 Platform Channel 调原生 API，和 RN Bridge 一样有线程切换和编解码开销。

**❌ 误区：JSI 让所有 Native 调用都变成同步的。**

✅ JSI 支持同步调用，但你**不应该**在主线程同步调用耗时操作。JSI 的正确用法是：计算型函数（加密、数学运算）用同步，I/O 操作（文件、网络）仍然异步。面试官喜欢听"我知道什么时候该同步、什么时候不该"。

**❌ 误区：Flutter 的 FFI 可以替代 Platform Channel。**

✅ FFI 能替代的是和 C/C++ 库的交互，不能替代和 Java/Kotlin/Swift 原生 API 的交互。Platform Channel 的价值是让 Dart 调用 Android/iOS 平台独有的系统 API（相机、HealthKit、通知），FFI 做不到这些。

**❌ 误区：通信机制选对了性能就够了。**

✅ 通信机制只解决"调用路径"的问题，真正的性能瓶颈往往在**调用频率**和**数据量**上。每秒调 1000 次 Native 方法，即使用 JSI，调用开销累计也是可观的。正确做法是：批量处理、降低频率、把重计算整体丢给 Native。

## 一句话总结

RN 的通信进化史是"从 JSON 异步到 JSI 同步"的一条路，Flutter 的通信体系是"Platform Channel（异步）+ FFI（同步）"的双轨制——两条路都在往同一个方向走：让跨语言调用消失。
