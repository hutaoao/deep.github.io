---
layout: post
title: "AndroidManifest.xml 完全解读——跨平台开发者必知的清单文件"
date: 2026-10-13
categories: ["原生扩展", "Android"]
tags: ["AndroidManifest", "清单合并", "组件注册", "intent-filter", "exported", "权限声明"]
description: >
  AndroidManifest 是应用向系统"自报家门"的唯一清单：组件、权限、入口、兼容信息全在里面；
  跨平台应用最终仍是标准 APK/AAB，集成插件时崩溃、不显示图标、权限过多，多半是它惹的祸。
---

## 一句话概括

`AndroidManifest.xml` 是 Android 应用的"身份证 + 说明书"——系统**只认清单里声明过的东西**：你有哪些四大组件、要申请哪些权限、哪个 Activity 是入口、依赖什么硬件，全靠它告诉系统。

对跨平台（Flutter / RN / 鸿蒙）开发者来说，它尤其关键：无论你用多"高级"的框架写业务，编译产物依然是一个标准 Android 应用，而系统只通过这份清单认识它。几乎所有"集成插件后运行时崩溃""装了不显示图标""权限莫名变多"的问题，根因都在清单。

一句话结论：**系统不信任代码里"偷偷做的事"，只信任清单里"明说的事"**。

## 核心知识点

### 1. 清单到底声明了什么：四大块

一份清单本质是向系统回答四个问题：我是谁、我有什么组件、我要什么权限、我兼容什么环境。

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.example.myapp">

    <application
        android:name=".MyApp"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/Theme.MyApp">

        <!-- 组件：Activity / Service / Receiver / Provider 都必须显式注册 -->
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

    <!-- 权限：向系统要能力 -->
    <uses-permission android:name="android.permission.CAMERA" />
    <!-- 硬件：声明依赖，影响 Google Play 的可见性 -->
    <uses-feature android:name="android.hardware.camera" android:required="false" />
</manifest>
```

面试常问"四大组件是什么"——`Activity`、`Service`、`BroadcastReceiver`、`ContentProvider`，**少一个没在清单注册的，系统就不让你启动它**（这也是安全模型的一部分）。

### 2. 身份与版本：namespace / package、versionCode、versionName

包名（namespace）是应用在系统中的唯一 ID，决定了 `R` 类、`BuildConfig` 的包路径、Intent 路由目标、以及 `/data/data/<包名>/` 数据目录。

```kotlin
// 现代 AGP（7.0+）推荐在 build.gradle.kts 里声明 namespace，
// 而不是（或叠加于）manifest 的 package 属性
android {
    namespace = "com.example.myapp"
    // ...
}
```

```xml
<!-- 早期写法：在 manifest 根节点声明 package，作为默认命名空间 -->
<manifest package="com.example.myapp"> ... </manifest>
```

要点：
- **AGP 7.0+ 后，官方建议用 `namespace`（在 gradle 里）取代 manifest 的 `package`**；manifest 里写 `.MainActivity` 这种相对名时，就是拿 namespace 当前缀拼成全类名。
- `versionCode` 是整数内部版本号，每次上架必须递增；`versionName` 是给人看的字符串。
- **包名一旦上架 Google Play 就不可更改**，改了会被当成全新应用。Flutter 项目创建默认 `com.example.xxx`，上线前务必先定好。
- Flutter 的 `pubspec.yaml` 里 `version: 1.0.0+1` 会自动映射到 `versionName`/`versionCode`，**别在 gradle 里又写一遍**，否则会被覆盖。

### 3. `<application>` 全局开关：theme / backup / 明文流量

`<application>` 是应用级配置中心，几个高频属性面试必考：

| 属性 | 作用 | 易踩的坑 |
|---|---|---|
| `android:theme` | 全局主题，控制启动闪屏、状态栏底色 | Flutter/RN 运行时改状态栏，但"闪屏阶段"样式全由它定 |
| `android:allowBackup` | 是否允许自动备份到云 | 默认 `true`，含敏感数据的应用应设为 `false` 或配 `backupRules` |
| `android:usesCleartextTraffic` | 是否允许 HTTP 明文 | **Android 9（API 28）起默认 `false`**，release 网络全挂多半是它 |
| `android:name` | 自定义 `Application` 子类 | 多个 SDK 都要初始化时，自己写一个 Application 统一接管 |

```xml
<application
    android:name=".MyApp"
    android:theme="@style/Theme.MyApp"
    android:allowBackup="false"
    android:usesCleartextTraffic="false"
    android:networkSecurityConfig="@xml/network_security_config" />
```

`usesCleartextTraffic` 是跨平台经典坑：debug 能联网、release 全失败，日志报 `CLEARTEXT communication not permitted`——根因就是它默认禁止明文。

### 4. `<activity>`：入口、exported、launchMode、深链接

Activity 必须显式注册，`exported` 和 `intent-filter` 是高频考点。

```xml
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:launchMode="singleTop"
    android:windowSoftInputMode="adjustResize">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

- **`exported`**：是否允许其它应用启动它。**从 Android 12（API 31）起，只要带了 `intent-filter`，`exported` 就必须显式写**，否则编译报错。这是跨平台项目升级 AGP/SDK 后最常见的"突然编不过"。
- **`launchMode`** 四种：`standard`（默认，每次新建）、`singleTop`（栈顶复用）、`singleTask`（清栈复用）、`singleInstance`（独占一栈）。Flutter 的 `MainActivity` 常设 `singleTop`，避免外部跳转（通知/深链接）每次都拉起新引擎。
- **深链接 / App Link**：用 `VIEW` + `BROWSABLE` + `data` 声明可处理的 URL，`autoVerify="true"` 时系统会校验 `assetlinks.json` 实现"无缝跳转"。

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="www.myapp.com" android:pathPrefix="/product" />
</intent-filter>
```

### 5. Service / Receiver / Provider：后台组件的当代约束

```xml
<service
    android:name=".MyMsgService"
    android:exported="false"
    android:foregroundServiceType="dataSync" />

<receiver
    android:name=".BootReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

- **Service**：Android 8（API 26）起后台启动受限，前台 Service 必须在通知栏显示通知，并声明 `foregroundServiceType`。推送（FCM）、后台定位、下载都依赖它。
- **Receiver**：Android 8 起**大部分隐式广播不再生效**，别指望注册个 `BOOT_COMPLETED` 之外的隐式广播就能在后台收到事件。
- **ContentProvider**：跨应用共享数据用，现代开发多由 `FileProvider`（分享文件）承担，需在清单声明并配 `file_paths.xml`。

### 6. 权限声明：`uses-permission` vs `permission`，以及 `uses-feature`

```xml
<!-- 向系统"要"权限（跨平台几乎只用这个） -->
<uses-permission android:name="android.permission.CAMERA" />
<!-- 自己"定义"一个权限给别人用（极少用） -->
<permission
    android:name="com.example.myapp.permission.MY_PERM"
    android:protectionLevel="signature" />
<!-- 声明硬件依赖；required=false 才不会无故丢掉没摄像头的用户 -->
<uses-feature android:name="android.hardware.camera" android:required="false" />
```

关键点：
- `uses-permission` = **请求**系统/其他应用的权限；`permission` = **定义**自己的权限。日常只用前者。
- `uses-feature` 的 `required="true"` 会让 Google Play 向缺该硬件的设备**隐藏你的应用**；只"可选"用的能力一定设 `false`。

### 7. 清单合并（Manifest Merger）：插件是怎么"混"进来的

跨平台项目里，主模块 + 每个第三方库/插件都自带清单，编译时由 AGP 合并成最终一份。

```text
合并优先级（高 → 低）：
  1. 主模块 AndroidManifest.xml
  2. 构建变体 src/<flavor>/AndroidManifest.xml
  3. 依赖库的 AndroidManifest.xml
```

冲突时用 `tools` 指令干预：

```xml
<manifest xmlns:tools="http://schemas.android.com/tools">
    <!-- 用主模块的 label 覆盖插件里的 -->
    <application
        android:label="@string/app_name"
        tools:replace="android:label" />
    <!-- 直接删掉某条被插件带进来的权限 -->
    <uses-permission
        android:name="android.permission.RECEIVE_BOOT_COMPLETED"
        tools:node="remove" />
</manifest>
```

查看合并结果：编译后在 `app/build/intermediates/merged_manifest/<变体>/AndroidManifest.xml`（不同 AGP 版本目录名略有差异，也可在 Android Studio 用 Build > Analyze APK 直接看最终 APK 的清单）。

## 其实你每天都在用

- **集成 `image_picker` 后拍照崩溃**：多半是忘了在清单声明 `CAMERA` 权限（Android 13+ 还需 `READ_MEDIA_IMAGES`）。
- **Android 12+ 升级后编译失败**：带 `intent-filter` 的 Activity 没写 `exported`，直接报 `missing EXPORTED flag`。
- **微信/支付宝分享回调收不到**：`WXEntryActivity` 必须在清单按固定路径 `.wxapi.WXEntryActivity` 注册，系统靠隐式 Intent 回调它。
- **点了推送/深链接却开新页面或没反应**：`launchMode` 与 `intent-filter`（VIEW + data）配置不对，或 App Link 的 `autoVerify` 没配 `assetlinks.json`。
- **release 包网络全挂**：`usesCleartextTraffic` 默认 `false`，调试靠 debug 构建的豁免，上线就露馅。
- **装了在桌面找不到图标**：入口 Activity 缺 `MAIN`+`LAUNCHER` 的 `intent-filter`，或被合并覆盖。
- **插件偷偷加了一堆权限**：合并后权限变多（如推送 SDK 带 `RECEIVE_BOOT_COMPLETED`），用 `tools:node="remove"` 剔除审计不通过的。

## 常见误解（FAQ）

**❌ 误区一："我是写 Flutter/RN 的，AndroidManifest 原生才用得到"**

错。无论框架多高级，产物仍是 APK/AAB，系统只认清单。插件（相机、推送、地图、分享）都会在合并时往清单塞组件和权限——你不看清单，就等于把应用的"系统登记信息"完全交给第三方，出事时毫无头绪。

**❌ 误区二："`exported` 默认 false，不写也行"**

不完全。API 30 及以下不带 `intent-filter` 的组件默认 `false`，但 **API 31+ 规定：只要组件带 `intent-filter`，`exported` 必须显式声明**，否则编译直接失败。入口 Activity、分享回调 Activity 都带 filter，升级 targetSdk 后这是第一道坎。

**❌ 误区三："清单里写了 `uses-permission` 就一定能用"**

只对了"普通权限"。`INTERNET`、`VIBRATE` 这类普通权限声明即授予；但 `CAMERA`、`READ_CONTACTS` 等**危险权限**只是"拿到申请资格"，真正能用还要在运行时弹窗征得用户同意（Android 6+）。Android 13+ 连通知、`READ_MEDIA_*` 媒体权限也要运行时申请。

**❌ 误区四："清单合并冲突只能删插件或改源码"**

大材小用。`tools:replace` 覆盖属性、`tools:remove` 删节点、`tools:node="merge"` 显式合并，三个指令基本能解决 90% 的合并冲突，完全不必动插件的代码。

**❌ 误区五："包名只是个名字，随时能改"**

包名是应用的唯一身份 ID。上架后再改，Google Play 会当成一个**全新应用**——评分、下载量、用户数据全部清零。上线前定好，且要和微信/支付等开放平台的登记包名完全一致，否则回调失败。

## 一句话总结

`AndroidManifest.xml` 是应用和系统之间的"契约"——你明说的系统才认，没说的系统一律拒绝；跨平台开发者不必手写每一个标签，但必须看得懂它、能在插件"自动混入"时做对取舍，这才是从"能跑"到"稳得上架"的分水岭。
