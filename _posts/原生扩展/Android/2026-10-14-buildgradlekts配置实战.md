---
layout: post
title: "build.gradle.kts 配置实战——跨平台开发者的 Gradle 构建指南"
date: 2026-10-14
categories: ["原生扩展", "Android"]
tags: ["build.gradle.kts", "AGP", "Kotlin DSL", "版本目录", "构建类型", "productFlavors"]
description: >
  build.gradle.kts 是现代 Android 构建的"总指挥"：SDK 版本、依赖、签名、混淆、多渠道全在这；
  跨平台项目 ninety percent 的原生报错都源于它，读懂它就能从"盲搜报错"变成"主动掌控"。
---

## 一句话概括

`build.gradle.kts` 是用 **Kotlin DSL** 写的构建脚本，定义了"怎么把代码打成 APK/AAB"的全部规则：编译哪个 SDK、依赖哪些库、怎么签名、是否混淆、输出哪几个渠道包。

对 Flutter / RN 开发者，它是绕不开的"原生黑箱"——`flutter run` 或 `run-android` 报的那堆红色 Gradle 错误，本质都在这两份脚本（项目级 + 模块级）和 `settings.gradle.kts` 里。掌握它，你从"试错-失败-搜"的被动循环，升级成"看错误就知道改哪"的主动模式。

一句话结论：**Gradle 不神秘，它就是一份带类型检查的"构建菜谱"，报错信息基本都直说了病因**。

## 核心知识点

### 1. 两份脚本 + 一个 settings：角色分清

```text
项目根/
├── settings.gradle.kts        # 仓库源、包含哪些 module
├── build.gradle.kts           # 项目级：声明插件版本（apply false）
└── app/
    └── build.gradle.kts       # 模块级：真正的编译配置主场
```

项目级只"声明"插件版本，不在本层应用：

```kotlin
// 项目根 build.gradle.kts
plugins {
    id("com.android.application") version "8.5.0" apply false
    id("org.jetbrains.kotlin.android") version "2.0.0" apply false
}
```

仓库集中到 `settings.gradle.kts`（AGP 7.0+ 推荐做法）：

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}
```

### 2. 模块级 `android {}`：SDK 三兄弟与 applicationId

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.myapp"   // 代码/R/BuildConfig 的包名
    compileSdk = 35                    // 用哪个 API 编译（建议追最新稳定版）

    defaultConfig {
        applicationId = "com.example.myapp" // 设备上/Play 里的唯一身份
        minSdk = 21
        targetSdk = 35                  // 当前新应用需 ≥35，2026-08-31 起需 ≥36
        versionCode = 1
        versionName = "1.0.0"
    }
}
```

| 字段 | 含义 | 面试要点 |
|---|---|---|
| `compileSdk` | 编译时用的 API 级别 | 不影响运行兼容性，建议追最新 |
| `minSdk` | 支持的最低系统版本 | 太低用不了新 API，AndroidX 也要求 ≥21 |
| `targetSdk` | 应用"面向"的版本 | **决定了哪些行为变更会作用于你**；Play 强制逐年追高 |
| `namespace` | 代码层包名（R 类路径） | AGP 7.0+ 取代 manifest 的 `package` |
| `applicationId` | 发布身份 | 上线后改了 = 全新应用，更新会失败 |

⚠️ **`namespace` 与 `applicationId` 不是一回事**：前者管代码寻址，后者管设备/商店身份。Flutter 的 `pubspec.yaml` 版本号会自动覆盖 `versionName/versionCode`，别在 gradle 里又写一遍。

### 3. `buildTypes`：debug / release 与签名

```kotlin
android {
    buildTypes {
        debug {
            isDebuggable = true
            applicationIdSuffix = ".debug"   // 同一手机装俩包互不冲突
        }
        release {
            isMinifyEnabled = true           // 开启 R8 压缩+混淆
            isShrinkResources = true         // 移除无用资源
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }

    signingConfigs {
        create("release") {
            // ❌ 别把密码写死进版本库
            storeFile = file("release.keystore")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }
}
```

- **R8** 自 AGP 3.4 起就是默认压缩/混淆工具（替代 ProGuard），`isMinifyEnabled = true` 即启用。
- release 包崩溃、debug 正常，八成是混淆把反射类/序列化模型重命名了——要在 `proguard-rules.pro` 里 `-keep`。

### 4. `dependencies`：implementation vs api，以及 KSP

```kotlin
dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.appcompat:appcompat:1.7.0")

    // api 会把依赖"透传"给上游模块；implementation 则隐藏
    // ❌ 滥用 api：改动会触发大量重编译
    // ✅ 默认全用 implementation，仅库模块要暴露公共 API 时才用 api
    api("com.squareup.retrofit2:retrofit:2.11.0")

    compileOnly("org.jetbrains:annotations:24.1.0")  // 仅编译期
    runtimeOnly("androidx.sqlite:sqlite-jdbc:2.4.0")  // 仅运行期

    // 注解处理：现代用 KSP 替代老旧的 kapt
    ksp("com.google.dagger:hilt-compiler:2.51.1")
}
```

| 配置 | 可见性 | 传递 | 场景 |
|---|---|---|---|
| `implementation` | 编译+运行，对外隐藏 | 否 | 绝大多数依赖 |
| `api` | 编译+运行，对外可见 | 是 | 库模块的公开依赖 |
| `compileOnly` | 仅编译 | 否 | 注解处理器、编译工具 |
| `runtimeOnly` | 仅运行 | 否 | 运行时实现 |

### 5. 版本目录（Version Catalog）：多模块不再版本地狱

```toml
# gradle/libs.versions.toml
[versions]
agp = "8.5.0"
kotlin = "2.0.0"
coreKtx = "1.13.1"

[libraries]
core-ktx = { module = "androidx.core:core-ktx", version.ref = "coreKtx" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

```kotlin
// 引用时自动生成类型安全访问器
dependencies {
    implementation(libs.core.ktx)
}
plugins {
    alias(libs.plugins.android.application) apply false
}
```

好处：版本单点维护、自动生成 `libs.xxx` 访问器、跨模块共享、能检测冲突。AGP 7.0+ 原生支持，是现代多模块项目的标配。

### 6. `productFlavors` + `flavorDimensions`：一套代码出多包

```kotlin
android {
    flavorDimensions += "env"
    productFlavors {
        create("dev") {
            dimension = "env"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
            // 配合 src/dev/ 下的配置/资源覆盖
        }
        create("prod") {
            dimension = "env"
        }
    }
}
```

生成的变体形如 `devDebug`、`devRelease`、`prodDebug`、`prodRelease`。跨平台常用它切 API 环境、图标、功能集（Flutter 用 `--flavor`，RN 配 `react-native-config`）。

### 7. 版本兼容铁三角：AGP × Gradle × JDK

```text
AGP 8.x  →  Gradle 8.x  →  JDK 17
Kotlin 2.x（K2 编译器）与 AGP 8.x 搭配
```

- AGP 8.0+ **强制 JDK 17**；`compileOptions` 与 `kotlinOptions` 的 JVM 目标也建议对齐 17。
- 版本不匹配会直接编译失败，错误信息里通常写明"requires Gradle x.y"。

```kotlin
android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
}

// 低 minSdk 想用 Java 8+ API（如 LocalDateTime）？开启 desugaring
dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")
}
```

## 其实你每天都在用

- **`flutter run` 报 AGP / JDK 版本不兼容**：红字里写着 requires Gradle x.y，照着升 `gradle-wrapper.properties` 即可。
- **改了 versionName 又被覆盖**：Flutter 的 `pubspec.yaml` 版本优先，去那改 `version: 1.0.0+1`。
- **release 崩溃、debug 正常**：R8 混淆干掉了插件的反射类，给 `proguard-rules.pro` 加 `-keep`。
- **接 Firebase 一堆错**：要加 `google-services` 插件 + `firebase-bom` 统一版本，且 `google-services.json` 放 `app/` 下。
- **dev/prod 两套环境**：用 `productFlavors` + `src/dev/` 覆盖 API 地址，告别硬编码。
- **依赖冲突报 `Conflict`**：跑 `./gradlew :app:dependencies` 看依赖树，用 BOM 或 `resolutionStrategy` 锁版本。
- **签名泄露风险**：`signingConfigs` 的密码走环境变量/CI 密钥，绝不进 Git。

## 常见误解（FAQ）

**❌ 误区一："`.gradle` 和 `.kts` 随便选，能编就行"**

能编，但体验差很多。`.kts` 是 **Kotlin DSL，带类型检查和 IDE 补全**，写错属性名编译期就报错；老 `.gradle`（Groovy）要运行时才暴露错误。Android Studio 新项目默认 `.kts`，官方也推荐迁移。

**❌ 误区二："`implementation` 和 `api` 没区别，都引依赖"**

区别在**是否把依赖透传给上游模块**。`api` 会让依赖"泄漏"出去，一旦它改动，所有依赖你的模块都得重编译，拖慢增量构建；`implementation` 则对外隐藏、编译更快。原则：**默认 implementation，只在库要对外暴露 API 时用 api**。

**❌ 误区三："`minSdk` 设越低，覆盖设备越多，越划算"**

事与愿违。太低（如 16/19）用不了新 API，且 **AndroidX / Jetpack 现在要求 `minSdk ≥ 21/23`**，很多新库直接不兼容。现代应用 `minSdk = 21`（覆盖 99%+ 设备）是性价比最佳点，没必要为了极少量老设备牺牲开发效率。

**❌ 误区四："混淆只是为了 APK 小一点"**

小体积只是副作用。混淆会**重命名类/方法/字段**——一旦你的代码或插件用反射、JSON 序列化（Gson/Moshi）、WebView 桥接按原名调用，release 就会 `NoSuchMethodException` / `ClassNotFoundException`。所以发布前务必在真机跑一遍 release 包。

**❌ 误区五："`targetSdk` 不用追，能跑就行"**

Google Play **强制逐年追高**：当前新应用/更新需 `targetSdk ≥ 35`（Android 15），2026-08-31 起需 `≥ 36`（Android 16），否则无法上架或更新。而且每升一级，该版本的行为变更（权限、后台限制、存储）都会作用到你的应用——建议每年追一级、逐级验证，别等被迫一次跨多级时 bug 叠 bug。

## 一句话总结

`build.gradle.kts` 不是玄学，而是带类型检查的"构建菜谱"：SDK 版本定边界、依赖定能力、签名混淆定发布质量、flavor 定多渠道；读懂它，跨平台项目里 90% 的"红色 Gradle 报错"都会从天书变成可定位的明确病因。
