---
tags: [android, 面试, 架构, UDF, A]
difficulty: A
frequency: medium
---

# 响应式架构 - UDF 单向数据流

## 问题

> 什么是单向数据流（UDF）？在 Android 中如何实现？它解决了什么问题？与双向数据绑定有什么区别？

## 核心答案

UDF（Unidirectional Data Flow）规定数据只能沿一个方向流动：UI 发出 Event → ViewModel 处理 → 产生新 State → UI 根据 State 渲染。这消除了状态不一致问题（多个数据源互相修改导致的 bug），使状态变化可追踪、可预测。与双向绑定（View↔ViewModel 互相修改）相比，UDF 牺牲了便利性换取了可靠性。

## 深入解析

### 原理层

**UDF 数据流向：**
```
┌──────────────────────────────────────┐
│                                      │
│  UI ──Event──→ ViewModel ──State──→ UI
│       (用户操作)    (处理逻辑)    (渲染)
│                                      │
└──────────────────────────────────────┘

单向循环：UI 不能直接修改 State，只能发送 Event 请求修改
```

**实现模式：**
```kotlin
// State（不可变）
data class SearchState(
    val query: String = "",
    val results: List<Item> = emptyList(),
    val isLoading: Boolean = false
)

// Event（用户意图）
sealed interface SearchEvent {
    data class QueryChanged(val text: String) : SearchEvent
    object Search : SearchEvent
    object ClearResults : SearchEvent
}

// ViewModel（状态持有者 + 事件处理器）
class SearchViewModel : ViewModel() {
    private val _state = MutableStateFlow(SearchState())
    val state: StateFlow<SearchState> = _state.asStateFlow()
    
    fun onEvent(event: SearchEvent) {
        when (event) {
            is SearchEvent.QueryChanged -> {
                _state.update { it.copy(query = event.text) }
            }
            is SearchEvent.Search -> {
                viewModelScope.launch {
                    _state.update { it.copy(isLoading = true) }
                    val results = repository.search(_state.value.query)
                    _state.update { it.copy(isLoading = false, results = results) }
                }
            }
            is SearchEvent.ClearResults -> {
                _state.update { it.copy(results = emptyList(), query = "") }
            }
        }
    }
}

// UI（纯渲染，不持有状态）
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    
    TextField(
        value = state.query,
        onValueChange = { viewModel.onEvent(SearchEvent.QueryChanged(it)) }
    )
    if (state.isLoading) CircularProgressIndicator()
    LazyColumn { items(state.results) { ItemRow(it) } }
}
```

**UDF vs 双向绑定：**

| 维度 | UDF | 双向绑定 (DataBinding) |
|------|-----|----------------------|
| 数据流向 | 单向循环 | View ↔ ViewModel 互相修改 |
| 状态源 | 单一（StateFlow） | 分散（多个 Observable） |
| 可预测性 | 高（状态变化可追踪） | 低（谁改了状态不确定） |
| 调试 | 容易（log State 变化） | 困难（回调链复杂） |
| 代码量 | 多（需要 Event 类） | 少（XML 直接绑定） |
| 适合场景 | 复杂交互 | 简单表单 |

**Compose 天然 UDF：**
- Compose 的 State hoisting 模式就是 UDF
- Composable 函数是纯函数：`(State) → UI`
- 状态提升到 ViewModel，UI 只负责渲染和发送事件

### 实战层

**状态一致性问题示例：**
```kotlin
// ❌ 双向绑定的问题
// 用户输入触发搜索 → 搜索结果更新列表 → 列表滚动触发加载更多 → 加载更多修改列表
// → 列表变化触发 UI 更新 → UI 更新触发某个监听 → 监听修改了搜索词...
// 状态互相影响，难以追踪 bug

// ✅ UDF 的解法
// 所有变化都经过 ViewModel.onEvent()
// State 只有一个来源，每次变化都可以 log
```

**调试技巧：**
```kotlin
// 拦截所有状态变化
private val _state = MutableStateFlow(SearchState())
    .also { flow ->
        viewModelScope.launch {
            flow.collect { state -> 
                Log.d("State", "New state: $state")  // 状态变化全记录
            }
        }
    }
```

**与 Redux 的关系：**
- Redux: Action → Reducer → Store → View → Action
- Android UDF: Event → ViewModel → State → UI → Event
- 本质相同，Android 版本更轻量（不需要中间件、Store 等概念）

### 延伸问题

- [[MVI vs MVVM vs MVC对比与选型]]
- [[Compose重组原理与性能优化]]
- [[Flow冷流vs热流]]

## 记忆锚点

UDF = 数据单行道。UI 只能看 State 不能改 State，想改就发 Event 给 ViewModel，ViewModel 是唯一能改 State 的人。好处：出了 bug 只需要看 Event 日志就能复现。
