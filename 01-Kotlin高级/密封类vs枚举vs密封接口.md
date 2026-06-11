---
tags: [android, 面试, kotlin, 类型系统, A]
difficulty: A
frequency: medium
---

# 密封类 vs 枚举 vs 密封接口

## 问题

> Kotlin 中 sealed class、enum class 和 sealed interface 各自的特点是什么？在什么场景下选择哪种？

## 核心答案

enum 适合固定数量的同类型常量（无状态差异）；sealed class 适合有限子类型且各子类可携带不同数据（如 UI State、Result）；sealed interface 在 sealed class 基础上支持多继承，适合跨模块的类型约束。三者都支持 when 表达式穷举检查。

## 深入解析

### 原理层

**enum class：**
```kotlin
enum class Direction { NORTH, SOUTH, EAST, WEST }
// 编译为 final class，每个枚举值是 static final 单例
// 所有实例共享相同的属性结构
```

**sealed class：**
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}
// 子类可以有不同的属性和构造函数
// 子类必须定义在同一模块（Kotlin 1.5+）同一包下
```

**sealed interface（Kotlin 1.5+）：**
```kotlin
sealed interface UiEvent
sealed interface AnalyticsEvent

// 一个类可以实现多个 sealed interface
data class ButtonClick(val id: String) : UiEvent, AnalyticsEvent
```

**编译层面差异：**
- enum → 单个 final class + static 实例
- sealed class → abstract class + 子类，编译器记录所有直接子类
- sealed interface → interface + 子类，支持多继承

**when 穷举原理：**
编译器在编译时知道所有子类型（通过 `$sealedSubclasses` 元数据），如果 when 没有覆盖所有分支会报错。

### 实战层

| 场景 | 推荐 | 原因 |
|------|------|------|
| HTTP 方法 GET/POST/PUT | enum | 固定常量，无额外数据 |
| 网络请求结果 | sealed class | Success 带数据，Error 带异常 |
| UI 状态 | sealed class/interface | 各状态携带不同数据 |
| 跨模块事件定义 | sealed interface | 一个事件可属于多个分类 |
| 有限的配置选项 | enum | 可带属性但结构统一 |

**Android 实战模式：**
```kotlin
// MVI 中的 Intent/Action
sealed interface HomeAction {
    object Refresh : HomeAction
    data class Search(val query: String) : HomeAction
    data class ItemClick(val id: Long) : HomeAction
}

// 导航事件
sealed interface NavEvent {
    data class Navigate(val route: String) : NavEvent
    object Back : NavEvent
}
```

- **坑点**：sealed class 的子类如果是 data class，equals/hashCode 自动生成；如果是普通 class 需要手动处理
- **性能**：enum 比 sealed class 轻量（单例 vs 对象创建），高频场景优先 enum

### 延伸问题

- [[MVI vs MVVM vs MVC对比与选型]]
- [[协程异常处理机制]]

## 记忆锚点

enum = 固定常量集合（同构），sealed = 有限类型族（异构），sealed interface = 可多继承的类型族。
