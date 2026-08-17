---
layout: post
title: "iOS 调试与 Instruments：从 LLDB 到崩溃分析的全方位指南"
date: 2026-10-12
categories: ["原生扩展", "iOS"]
tags: [LLDB, Xcode断点, 崩溃分析, dSYM, Instruments, 内存泄漏]
description: >
  面试高频：LLDB 常用命令（po/p/bt）、Xcode 四类断点、崩溃日志与 dSYM 符号化、Instruments 四大利器（Time Profiler/Allocations/Leaks/Core Animation），一篇讲透 iOS 原生调试。
---

## 一句话概括

iOS 调试武器库有四位主角：**LLDB**（命令行调试器，运行时探对象、改值、看调用栈）、**Xcode 断点系统**（普通/异常/符号/条件断点，交互式暂停）、**崩溃日志 + dSYM 符号化**（把机器地址还原成源码行）、**Instruments**（性能与内存的量化分析仪）。跨端同学遇到原生崩溃、内存暴涨、列表卡顿时，最终都要回到这套工具上定位根因。

面试常问：**"po 和 p 有什么区别""异常断点干嘛用""崩溃日志怎么符号化""内存泄漏怎么用 Instruments 查""列表卡顿怎么分析"**。核心心法一句话：**先靠异常类型和崩溃线程缩小范围，再用 LLDB 看对象状态，用 Instruments 把'感觉卡'变成'哪行吃 CPU/哪块内存没释放'的可量化结论**。

## 核心知识点

### 1. LLDB：运行时探针

最常用的几条：

```lldb
po self.view                 // 打印对象 description（最常用）
po tableView.visibleCells.count
p self.view.frame            // 打印表达式类型和值，适合基础类型/结构体
p sizeof(CGFloat)

expr self.titleLabel.text = "新标题"     // 运行时改值，不用重新编译
expr tableView.isHidden = true

bt                           // 当前线程调用栈
bt all                       // 所有线程
frame select 3               // 跳到调用栈第 3 帧
frame variable               // 看当前帧局部变量

breakpoint set -n "viewDidLoad"                 // 符号断点
breakpoint set -f "VC.swift" -l 42              // 文件行断点
breakpoint set -n "tableView:didSelectRowAtIndexPath:" -c "row == 0"  // 条件断点
```

`po` vs `p`：**`po` 调对象的 `description` 出描述性字符串，适合对象；`p` 直接给表达式的类型+值，适合 `CGRect`/`Int`/`BOOL` 等基础类型**。两者对对象都能打印，但 `p` 还会带类型标注。

调试 Auto Layout 时常用：

```lldb
expr -l objc -O -- [self.view _autolayoutTrace]   // 看约束冲突/不明确
po [self.window recursiveDescription]             // 看视图层级
```

### 2. Xcode 四类断点

| 类型 | 用途 | 面试考点 |
|---|---|---|
| 普通断点 | 某行暂停，可加条件/忽略次数 | 条件表达式只对 true 触发 |
| **异常断点** | 抛异常即停，停在抛出那行 | 定位崩溃原点的首选 |
| **符号断点** | 某方法被调用即停（如 `-[UIViewController viewDidLoad]`） | 不用找所有调用点 |
| 断点 Action | 触发时执行 LLDB/Log/Sound，可"自动继续" | 不暂停也能打日志追踪 |

异常断点是排查原生崩溃的第一道闸：创建后，NSException 一抛就停在**抛出那一行**，而不是崩在调用栈顶端让你猜。

### 3. 崩溃日志与 dSYM 符号化

未符号化的崩溃长这样（全是地址）：

```text
0   MyApp   0x0000000102a8c234   0x10289c000 + 2030132
1   UIKitCore  0x00000001c3a4718c   ...
```

符号化后：

```text
0   MyApp   $s6MyApp11ViewControllerC11viewDidLoadyyF + 68
1   UIKitCore   -[UIViewController _sendViewDidLoadWithColor:] + 180
```

**关键规则**：崩溃日志必须和**生成它的那个二进制完全匹配**的 dSYM 才能符号化——所以每个发布版本都要留好 dSYM（Xcode Organizer 里勾选"上传符号"最省事）。手动符号化用 `atos`：

```bash
atos -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp \
     -arch arm64 -l 0x10289c000 0x0000000102a8c234
# 输出：-[ViewController viewDidLoad] + 68
```

常见 `Exception Type`：

- `EXC_BAD_ACCESS (SIGSEGV)`：访问已释放/非法内存（野指针）
- `SIGABRT`：主动 abort（未捕获 NSException、assert 失败）
- `SIGKILL`：系统杀（内存超限、看门狗超时）
- `EXC_BREAKPOINT`：Swift 运行时错误（如强制解包 nil）

### 4. Instruments 四大利器

| 工具 | 查什么 | 关键指标 / 技巧 |
|---|---|---|
| **Time Profiler** | CPU 卡在哪 | 勾 `Hide System Libraries` + `Invert Call Tree`，找自己代码里最吃 CPU 的方法 |
| **Allocations** | 内存增长 / 泄漏线索 | `Persistent Bytes`、`# Persistent`；A→B→A 循环后若持续涨，有泄漏 |
| **Leaks** | 确认内存泄漏 | 红色旗帜标出泄漏对象，看引用路径里"多余的强持有" |
| **Core Animation** | 渲染掉帧 | 勾 `Color Blended Layers`（红=混合）、`Color Offscreen-Rendered`（黄=离屏渲染） |

Time Profiler 实战（列表滚动卡）：

```swift
func tableView(_ tv: UITableView, cellForRowAt ip: IndexPath) -> UITableViewCell {
    let cell = tv.dequeueReusableCell(withIdentifier: "c", for: ip) as! Cell
    let item = data[ip.row]
    // ❌ 在主线程同步解码大图 / 复杂布局，滚动必卡
    // ✅ 异步解码后回主线程赋值
    DispatchQueue.global().async {
        let img = decodeResized(item.imagePath, target: cellSize)
        DispatchQueue.main.async { cell.thumb.image = img }
    }
    return cell
}
```

### 5. 真机调试要点（iOS 16+）

- 首次真机运行要在设备上 **信任开发者证书**（设置 → 通用 → VPN 与设备管理）
- **iOS 16+ 需开启开发者模式**（设置 → 隐私与安全性 → 开发者模式），否则跑不起来
- 免费账号每个 App ID 最多装 3 台设备，Provisioning Profile 7 天过期；付费账号（¥688/年）无此限制
- Xcode 11+ 支持 **Wi-Fi 无线调试**：USB 连一次后，Devices and Simulators 里勾 "Connect via network"

## 其实你每天都在用

- **`po` 看某个 label 的文字 / 数组个数**：调试时最常用的"打印"
- **异常断点定位 `fatalError` / 越界崩溃**：一抛就停，不用翻调用栈猜
- **符号断点 `-[UIViewController viewDidLoad]`**：想看所有页面初始化顺序时
- **Xcode Memory Graph Debugger**：点一下看对象引用环，找 VC 没释放的元凶
- **Time Profiler 查列表卡顿**：勾 Hide System Libraries 直击自己代码的耗时方法
- **Allocations 看"进页面→返回"内存是否回落**：不回落=泄漏
- **Leaks 红色旗帜**：直接告诉你哪个对象泄漏、被谁强持有
- **Core Animation 的 Color Blended Layers**：一眼看出哪些视图在无效混合、该设 `opaque`

## 常见误解（FAQ）

**❌ 误区一："po 和 p 差不多，随便用"**

有区别：`po` 调 `description` 出**描述性字符串**，适合看对象内容；`p` 出**类型+值**，更适合 `CGRect`/`CGPoint`/`Int`/`BOOL` 这类基础类型。看一个 `UIView` 的 frame 用 `p self.view.frame` 直接出结构体；用 `po` 出的是 description 摘要。面试能说清这点的，显得真用过 LLDB。

**❌ 误区二："崩溃日志看不懂，只能丢给第三方平台"**

先抓重点：① 看 `Exception Type`（EXC_BAD_ACCESS=野指针，SIGABRT=未捕获异常，SIGKILL=被系统杀）；② 看 `Thread 0 Crashed` 的调用栈——符号化后崩溃点一目了然；③ 确认 dSYM 的 UUID 和日志匹配，不匹配就符号化失败（这是"只有系统库栈"的头号原因）。掌握这三点，八成崩溃能自己定位。

**❌ 误区三："Instruments 里 system 库占大头，没法看自己的代码"**

Time Profiler 的 Call Tree 勾上 **`Hide System Libraries`** 就只显示你的代码，再勾 `Invert Call Tree` 把最深帧放最前，一眼看出"哪行最吃 CPU"。Allocations 重点看 `Persistent Bytes` 是否随页面进出持续增长。不是工具不行，是没开对过滤项。

**❌ 误区四："看到内存涨就是泄漏，用 Leaks 查就行"**

内存增长 ≠ 泄漏。缓存、图片解码、临时对象都会涨，关键看**是否回落**。正确姿势：用 Allocations 跑"进页面→返回"循环，若 `# Persistent` 持续涨说明有对象没释放；再用 Leaks 或 Memory Graph 看具体是哪个强引用环（闭包捕获 self、NSTimer 没 invalidate、通知没移除）。Leaks 报红是"确诊"，Allocations 趋势是"初筛"。

**❌ 误区五："条件断点/符号断点在 Swift 闭包、Extension 里不灵"**

多半是配置问题：闭包内断点要加在**具体行**；Swift Extension 的方法断点可能在文件选择器里定位不到，建议用符号断点写全签名；调试不触发先确认没被全局禁用、Debug 优化级别是 `-O0`（Release 优化会内联掉代码）。不是 LLDB 不支持，是优化或配置拦住了。

## 一句话总结

iOS 调试不是"漫无目的试"，而是一场**基于证据的推理**：LLDB 的 `po`/`p`/`bt` 让你在运行时看清对象状态，Xcode 异常断点把你直接送到崩溃原点，dSYM 把地址还原成源码行，Instruments 把"感觉卡/内存大"变成"哪行吃 CPU、哪块没释放"的量化结论。记住——**异常类型已经给了一半答案，调用栈指出了位置，Instruments 标出了热点**，Xcode 工具链就是你排查原生问题的侦探工具箱。
