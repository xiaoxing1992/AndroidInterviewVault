---
tags: [android, 面试, jetpack, navigation, A]
difficulty: A
frequency: medium
---

# Navigation 组件原理与深链接

## 问题

> Jetpack Navigation 组件的核心原理是什么？Fragment 切换是如何实现的？Deep Link 是怎么工作的？

## 核心答案

Navigation 通过 NavController 管理导航图（NavGraph）中的目的地（Destination）切换。Fragment 目的地通过 FragmentNavigator 执行 FragmentTransaction 实现切换，NavController 维护一个回退栈（BackStack）管理导航历史。Deep Link 通过在 NavGraph 中声明 URI 模式，系统 Intent 匹配后由 NavController 直接导航到目标 Destination。

## 深入解析

### 原理层

**核心架构：**
```
NavHostFragment (容器)
  └── NavController (控制器)
        ├── NavGraph (导航图，包含所有 Destination)
        ├── Navigator<Destination> (执行实际导航)
        │   ├── FragmentNavigator → FragmentTransaction
        │   ├── ActivityNavigator → startActivity
        │   └── DialogFragmentNavigator → show()
        └── BackStack (回退栈)
```

**NavController.navigate() 流程：**
```kotlin
// 1. 解析目的地
val node = graph.findNode(destinationId)

// 2. 处理 NavOptions（动画、popUpTo、singleTop）
// 3. 调用对应 Navigator 执行导航
navigator.navigate(destination, args, navOptions, navigatorExtras)

// 4. 更新 BackStack
addBackStackEntry(destination, args)
```

**FragmentNavigator 实现：**
```java
class FragmentNavigator : Navigator<Destination>() {
    override fun navigate(destination: Destination, args: Bundle?, 
                         navOptions: NavOptions?, extras: Extras?): NavDestination? {
        val fragment = instantiateFragment(destination.className)
        fragment.arguments = args
        
        val ft = fragmentManager.beginTransaction()
        ft.setCustomAnimations(...)  // 设置动画
        ft.replace(containerId, fragment)  // 替换 Fragment
        ft.setPrimaryNavigationFragment(fragment)
        ft.addToBackStack(generateBackStackName())
        ft.commit()
        
        return destination
    }
}
```

**Deep Link 工作机制：**
```xml
<!-- nav_graph.xml -->
<fragment android:id="@+id/detailFragment">
    <deepLink app:uri="myapp://detail/{id}" />
    <!-- 编译时生成 intent-filter 到 AndroidManifest -->
</fragment>
```

```kotlin
// 处理流程
// 1. 系统 Intent 匹配 → 启动 Activity
// 2. Activity 中 NavController 检查 Intent
navController.handleDeepLink(intent)
// 3. 解析 URI，匹配 NavGraph 中的 deepLink 声明
// 4. 构建完整的回退栈（从 startDestination 到目标）
// 5. 导航到目标 Destination，传入 URI 参数
```

**Safe Args 原理：**
- Gradle 插件在编译时根据 NavGraph XML 生成 Directions 类和 Args 类
- 类型安全的参数传递，避免 key 拼写错误

### 实战层

- **单 Activity 架构**：Navigation 推动了单 Activity + 多 Fragment 的架构模式
- **嵌套导航图**：复杂 App 用 `<navigation>` 嵌套实现模块化导航
- **Compose Navigation**：`NavHost` + `composable()` DSL，底层仍是 NavController
- **坑点**：Fragment 重建时 View 重建但 ViewModel 存活，注意 View 绑定的生命周期
- **回退栈管理**：`popUpTo` + `inclusive` 控制回退行为，避免栈中积累过多 Fragment

```kotlin
// Compose Navigation 示例
NavHost(navController, startDestination = "home") {
    composable("home") { HomeScreen(navController) }
    composable("detail/{id}") { backStackEntry ->
        DetailScreen(id = backStackEntry.arguments?.getString("id"))
    }
}
```

### 延伸问题

- [[组件化路由方案-ARouter原理]]
- [[Lifecycle感知组件实现]]

## 记忆锚点

Navigation = NavController（指挥官）+ NavGraph（地图）+ Navigator（司机）。Deep Link 就是给目的地贴个 URL 标签，外部可以直接跳过去。
