---
tags: [android, 面试, 架构, 插件化, S]
difficulty: S
frequency: medium
---

# 插件化原理 - ClassLoader 方案

## 问题

> Android 插件化的核心原理是什么？如何加载未安装 APK 中的类和资源？有哪些主流方案？为什么现在插件化逐渐式微？

## 核心答案

插件化核心是通过自定义 ClassLoader 加载插件 APK 中的 dex 文件，通过 AssetManager 反射添加插件资源路径来加载资源，通过 Hook AMS 或使用占坑 Activity 来绕过系统对未注册组件的校验。主流方案有 RePlugin（360）、VirtualApk（滴滴）、Shadow（腾讯）。随着 App Bundle、模块化、动态交付的成熟，插件化逐渐被替代。

## 深入解析

### 原理层

**三大核心问题：**

**1. 类加载 — DexClassLoader：**
```kotlin
// 加载插件 APK 中的类
val pluginClassLoader = DexClassLoader(
    pluginApkPath,           // 插件 APK 路径
    optimizedDir,            // dex 优化输出目录
    nativeLibDir,            // so 库路径
    parentClassLoader        // 父 ClassLoader
)

// 方案A：合并 dexElements（单 ClassLoader）
// 将插件的 dexElements 合并到宿主的 PathClassLoader 中
val hostDexElements = getField(hostClassLoader, "pathList.dexElements")
val pluginDexElements = getField(pluginClassLoader, "pathList.dexElements")
val merged = Array.newInstance(Element::class.java, host.size + plugin.size)
// 设置回 hostClassLoader

// 方案B：多 ClassLoader（隔离）
// 每个插件独立 ClassLoader，通过委托机制互相访问
```

**2. 资源加载 — AssetManager：**
```kotlin
// 反射调用 AssetManager.addAssetPath 添加插件资源
val assetManager = AssetManager::class.java.newInstance()
val addAssetPath = AssetManager::class.java.getDeclaredMethod("addAssetPath", String::class.java)
addAssetPath.invoke(assetManager, pluginApkPath)

// 创建插件的 Resources 对象
val pluginResources = Resources(assetManager, hostResources.displayMetrics, hostResources.configuration)
```

**3. 组件生命周期 — Hook/占坑：**
```
方案A: Hook AMS（动态代理）
  startActivity(pluginActivity)
  → Hook 点: IActivityManager.startActivity()
  → 替换 Intent 为占坑 StubActivity
  → AMS 校验通过（StubActivity 已注册）
  → Hook 点: Handler.mCallback (ActivityThread.H)
  → 替换回 pluginActivity
  → 反射创建插件 Activity 实例

方案B: 代理模式（Shadow）
  → 插件 Activity 不继承 android.app.Activity
  → 继承自 SDK 提供的 ShadowActivity（普通类）
  → 宿主的 ContainerActivity 持有 ShadowActivity
  → 生命周期由 ContainerActivity 转发
```

**各方案对比：**

| 方案 | 原理 | Hook 程度 | 兼容性 |
|------|------|-----------|--------|
| RePlugin | 占坑 + ClassLoader | 极少 Hook | 高 |
| VirtualApk | Hook AMS + 合并资源 | 中等 | 中 |
| Shadow | 代理模式 + 编译时转换 | 零 Hook | 最高 |
| DroidPlugin | 完全 Hook Framework | 大量 Hook | 低 |

### 实战层

**为什么插件化式微：**
- Android 9+ 限制反射访问隐藏 API（@hide）
- Google Play 禁止动态代码加载
- App Bundle + Dynamic Feature Module 提供官方动态交付
- 组件化 + 模块化已能满足大部分解耦需求
- 维护成本高，每个 Android 版本都可能 break

**仍有价值的场景：**
- 超级 App（微信、支付宝）的业务隔离
- 热修复（本质是类替换，插件化的子集）
- 国内应用市场分发（不受 Google Play 限制）

**Shadow 的零 Hook 方案：**
```kotlin
// 编译时：Gradle 插件将插件代码中的
//   android.app.Activity → com.tencent.shadow.core.runtime.ShadowActivity
//   getResources() → 代理方法
// 运行时：宿主 ContainerActivity 管理生命周期
// 优势：不依赖任何系统隐藏 API
```

### 延伸问题

- [[热修复原理-类加载与底层替换]]
- [[组件化路由方案-ARouter原理]]
- [[模块化依赖管理策略]]

## 记忆锚点

插件化三板斧：ClassLoader 加载类、AssetManager 加载资源、Hook/占坑骗过 AMS。现在式微因为 Google 封堵反射 + 官方有了 Dynamic Feature。
