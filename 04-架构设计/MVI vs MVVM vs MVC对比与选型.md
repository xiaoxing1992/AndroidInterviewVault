---
tags: [android, 面试, 架构, mvi, A]
difficulty: A
frequency: high
---

# MVI vs MVVM vs MVC 对比与选型

## 问题

> MVC、MVP、MVVM、MVI 各自的核心思想是什么？在 Android 中各有什么优缺点？什么场景下选择 MVI？

## 核心答案

MVC 将逻辑放在 Controller（Android 中 Activity 承担过多职责）；MVP 通过 Presenter 接口解耦 View 和 Model；MVVM 通过数据绑定/观察实现 View 自动更新；MVI 在 MVVM 基础上引入单向数据流（Intent→Model→View），用不可变 State 和 Reducer 管理状态。MVI 适合状态复杂、需要可预测性和可调试性的场景。

## 深入解析

### 原理层

**架构演进对比：**

```
MVC:  View ←→ Controller ←→ Model
      (Activity 同时是 View 和 Controller，职责混乱)

MVP:  View ←→ Presenter ←→ Model
      (接口隔离，但 Presenter 与 View 1:1 绑定，接口爆炸)

MVVM: View ──observe──→ ViewModel ←→ Model
      (数据驱动，View 被动更新，但多个 LiveData 状态难以同步)

MVI:  View ──Intent──→ ViewModel ──State──→ View
      (单向数据流，单一状态源，可预测)
```

**MVI 核心概念：**
```kotlin
// Intent（用户意图）
sealed interface HomeIntent {
    object LoadData : HomeIntent
    data class Search(val query: String) : HomeIntent
}

// State（不可变 UI 状态）
data class HomeState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val error: String? = null
)

// Reducer（纯函数，旧状态 + 事件 → 新状态）
fun reduce(state: HomeState, result: Result): HomeState = when (result) {
    is Result.Loading -> state.copy(isLoading = true)
    is Result.Success -> state.copy(isLoading = false, items = result.data)
    is Result.Error -> state.copy(isLoading = false, error = result.message)
}
```

**MVI ViewModel 实现：**
```kotlin
class HomeViewModel : ViewModel() {
    private val _state = MutableStateFlow(HomeState())
    val state: StateFlow<HomeState> = _state.asStateFlow()
    
    fun processIntent(intent: HomeIntent) {
        when (intent) {
            is HomeIntent.LoadData -> loadData()
            is HomeIntent.Search -> search(intent.query)
        }
    }
    
    private fun loadData() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            val result = repository.getData()
            _state.update { it.copy(isLoading = false, items = result) }
        }
    }
}
```

**各架构对比表：**

| 维度 | MVC | MVP | MVVM | MVI |
|------|-----|-----|------|-----|
| 数据流 | 双向 | 双向 | 单向观察 | 严格单向 |
| 状态管理 | 分散 | 分散 | 多 LiveData | 单一 State |
| 可测试性 | 低 | 高 | 高 | 最高 |
| 样板代码 | 少 | 多（接口） | 中 | 中 |
| 调试性 | 低 | 中 | 中 | 高（状态可追溯） |
| 学习曲线 | 低 | 中 | 中 | 高 |

### 实战层

**选型建议：**
- **简单页面**（设置页、关于页）→ MVVM 足够
- **复杂交互**（购物车、表单、聊天）→ MVI（状态多且相互关联）
- **团队协作**（大型项目）→ MVI（约束强，代码风格统一）
- **遗留项目**→ 逐步从 MVP/MVVM 迁移到 MVI

**MVI 的优势场景：**
- 状态回放/时间旅行调试
- 多个 UI 组件共享同一状态
- 需要乐观更新 + 回滚
- 复杂的状态转换逻辑

**MVI 的代价：**
- 简单页面过度设计
- State 对象频繁创建（data class copy）
- 一次性事件（Toast/导航）需要额外处理（Side Effect）

```kotlin
// Side Effect 处理
sealed interface HomeSideEffect {
    data class ShowToast(val message: String) : HomeSideEffect
    data class Navigate(val route: String) : HomeSideEffect
}

// 用 Channel 发送一次性事件
private val _sideEffect = Channel<HomeSideEffect>(Channel.BUFFERED)
val sideEffect = _sideEffect.receiveAsFlow()
```

### 延伸问题

- [[响应式架构-UDF单向数据流]]
- [[Clean Architecture在Android的落地]]
- [[LiveData vs Flow选型]]

## 记忆锚点

MVC→MVP→MVVM→MVI 是逐步约束数据流方向的过程。MVI = 单一状态 + 单向数据流 + 纯函数 Reducer，像 Redux 在 Android 的翻版。
