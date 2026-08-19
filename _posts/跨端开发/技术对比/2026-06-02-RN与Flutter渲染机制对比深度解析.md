---
layout: post
title: "RN vs Flutter 渲染机制面试指南深度解析：Yoga、Skia 和 Fabric 到底谁快"
description: >
  对比题：RN（Yoga 布局 + 原生视图）与 Flutter（Skia 自绘）的渲染链路与 Fabric 新架构。
  讲清谁更快、为什么，掌握后能应对渲染性能追问。
date: 2026-06-02
categories: [跨端开发, 技术对比]
tags: [React Native, Flutter, Yoga, Fabric, Skia, 面试]
---

## 一句话概括

RN 用 Yoga 引擎算好布局后交给**原生控件**渲染，Flutter 用 Skia/Impeller 引擎**自绘所有像素**——面试时不要只说"Flutter 自绘所以快"，要能讲出两条渲染管线每一步在干什么。

## 核心知识点

### 1. RN 的渲染链路：JS 描述 → Yoga 布局 → 原生 View

{% raw %}
```javascript
// JS 侧写的就是 React，但你每写一个 <View> 就会生成一个原生 UIView/ViewGroup
<View style={{ flexDirection: 'row', padding: 16 }}>
  <Text style={{ fontSize: 18 }}>标题</Text>
  <Text style={{ color: 'gray' }}>副标题</Text>
</View>
```
{% endraw %}

链路：JS 线程做 diff → Fabric/C++ 线程用 Yoga 做 Flexbox 布局 → 主线程创建/更新原生 View。**每一步都要跨线程通信。**

### 2. Flutter 的渲染链路：Widget → RenderObject → Layer → GPU

```dart
// 同一个卡片在 Flutter 里不需要创建任何原生 View
Container(
  padding: EdgeInsets.all(16),
  child: Row(children: [
    Text('标题', style: TextStyle(fontSize: 18)),
    Text('副标题', style: TextStyle(color: Colors.grey)),
  ]),
)
```

链路全在 Dart/Engine 内部：Widget (配置) → Element (生命周期) → RenderObject (布局+绘制) → Layer → GPU。**零跨语言调用。**

### 3. Yoga 布局引擎：Flexbox 的 C++ 实现

```cpp
// Yoga 的核心：O(n) 单次遍历完成所有布局计算
void YGNodeCalculateLayout(node, availableWidth, availableHeight) {
  applyPaddingAndBorder(node);
  for (auto child : node.children) {
    YGNodeCalculateLayout(child, childWidth, childHeight); // 递归
    child.setPosition(top, left); // 设最终位置
  }
}
```

Yoga 只负责位置和尺寸，不参与绘制——真正画 View 的是原生 UIKit/Android View 系统。

### 4. Impeller 替代 Skia 的意义：消灭着色器编译卡顿

Skia 需要**运行时编译着色器**——动画/页面首次出现时会卡顿（shader jank）。Impeller 改用**预编译（AOT）着色器**：

| | Skia | Impeller |
|---|---|---|
| 着色器编译 | 运行时 JIT → jank | 构建时 AOT → 无 jank |
| GPU 后端 | OpenGL/Metal | Vulkan/Metal |
| 首帧性能 | 有编译抖动 | 完全确定 |

**面试加分**：能说出"Impeller 解决的是 shader jank 问题，不是替代 Skia 而是针对移动端重新实现"。

### 5. RepaintBoundary：Flutter 的重绘隔离机制

```dart
RepaintBoundary(
  child: ComplexAnimationWidget(), // 这个子树的 repaint 不会影响外部
)
// 等价于：给这个子树拍一张"离屏快照"，只有它变化时才重绘这部分
```

过度使用会耗尽 GPU 内存，太少则重绘面积过大。经验：**每个长列表 item 外包一层，每个独立动画组件包一层，静态页面不需要。**

## 其实你每天都在用

- **CSS Flexbox** 就是 Yoga 的"老师"——你写 RN 布局时其实在写 Flexbox，只是背后是 C++ 在算
- **Chrome DevTools 的 Paint flashing** 标出的绿色闪块 = Flutter 里需要 RepaintBoundary 的地方
- **iOS 的 `shouldRasterize`** 和 RepaintBoundary 做的是同一件事——离屏缓存避免重复绘制
- **Web 的 `will-change: transform`** 和 Flutter 的 RepaintBoundary 都是"告诉渲染引擎这里会频繁变，给我一个独立层"
- **视频播放器渲染**——就是典型的"自绘引擎"思维：不依赖系统 UI 控件，自己把每一帧画到屏幕

## 常见误解（FAQ）

**❌ 误区：Flutter 不用原生组件所以包体积大，RN 用原生组件所以包体积小。**

✅ Flutter 大是因为打包了 Skia/Impeller 引擎（~5MB），RN 大是因为打包了 JS 引擎（Hermes ~3MB）+ React 框架。空应用的包体积差距只有 2-3MB，不是决定因素。

**❌ 误区：RN 的 `useNativeDriver: true` 让动画和 Flutter 一样快。**

✅ Native Driver 只支持 transform 和 opacity，不支持任何布局属性（width/height/left/top）。Flutter 可以动画任意属性且都在 UI 线程 vsync 回调中运行——灵活性不在一个量级。RN 想同时动画布局+透明度必须拆成两个动画。

**❌ 误区：Flutter 长列表性能碾压 RN 是因为语言快。**

✅ Flutter ListView 快是因为每个 item 不创建原生 View——没有 UIView/ViewGroup 的创建和回收开销。RN FlatList 即使优化到极致，每个 item 还是要创建一个原生 View。这是架构差异，不是语言差异。

**❌ 误区：RN 新架构的 Fabric 渲染器后，渲染性能和 Flutter 一样了。**

✅ Fabric 优化了 JS ↔ Shadow Tree 之间的通信（同步而非异步），但最终还是要创建原生 View，仍然受平台渲染管线的限制。Flutter 的 DisplayList → GPU 路径更短，极端复杂场景（5000+ item 列表）的差距依然存在。

## 一句话总结

RN 的渲染路径是 JS → Yoga → 原生 View → GPU（中间每一跳都有开销），Flutter 的路径是 Dart → RenderObject → DisplayList → GPU（全是内部跳转）——面试官不是要你记住每层名字，是要你理解"跨语言调用次数"这个根本差异。

