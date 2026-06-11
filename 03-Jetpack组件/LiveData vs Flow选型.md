---
tags: [android, 面试, jetpack, livedata, flow, A]
difficulty: A
frequency: high
---

# LiveData vs Flow 选型

## 问题

> LiveData 和 Kotlin Flow 各自的优缺点是什么？在什么场景下选择哪个？Google 为什么推荐用 Flow 替代 LiveData？

## 核心答案

LiveData 生命周期感知、主线程分发、简单易用，但不支持背压、操作符有限、与 Kotlin 协程生态割裂。Flow 功能强大（丰富操作符、背压支持、线程切换灵活），配合 `repeatOnLifecycle` 可以实现生命周期感知。Google 推荐 Flow 因为它是 Kotlin-first 的统一响应式方案，LiveData 仅在极简场景或 Java 项目中仍有价值。

## 深入解析

### 原理层

**对比表：**

| 特性 | LiveData | StateFlow | SharedFlow |
|------|----------|-----------|------------|
| 生命周期感知 | 内置 | 需要 repeatOnLifecycle | 需要 repeatOnLifecycle |
| 线程 | 主线程分发 | 任意线程 | 任意线程 |
| 初始值 | 可选 | 必须 | 无 |
| 去重 | 不去重 | distinctUntilChanged | 不去重 |
| 背压 | 无（总是最新值） | DROP_OLDEST | 可配置 |
| 操作符 | map/switchMap/mediator | 完整 Flow 操作符 | 完整 Flow 操作符 |
| 多观察者 | 各自独立回调 | 共享上游 | 共享上游 |
| 数据粘性 | 有（新观察者收到最后值） | 有（replay=1） | 可配置 |

**LiveData 的问题：**
```kotlin
// 1. 数据倒灌（粘性事件）
// 新 Fragment observe 时会收到上一次的值
viewModel.event.observe(this) { handleEvent(it) }  // 可能重复处理

// 2. 操作符不足
// 需要 MediatorLiveData 组合多个源，代码冗长
val combined = MediatorLiveData<Pair<A, B>>()
combined.addSource(liveDataA) { a -> combined.value = Pair(a, liveDataB.value) }
combined.addSource(liveDataB) { b -> combined.value = Pair(liveDataA.value, b) }

// 3. 转换在主线程
liveData.map { heavyTransform(it) }  // 在主线程执行，可能卡顿
```

**Flow 的正确收集方式：**
```kotlin
// ✅ 推荐：repeatOnLifecycle
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            updateUI(state)
        }
    }
}

// ✅ 简写（单个 Flow）
viewModel.uiState
    .flowWithLifecycle(lifecycle, Lifecycle.State.STARTED)
    .onEach { updateUI(it) }
    .launchIn(lifecycleScope)

// ❌ 错误：直接在 lifecycleScope.launch 中 collect
// Activity 进入后台时不会停止收集，浪费资源
lifecycleScope.launch {
    viewModel.uiState.collect { ... }  // 后台仍在收集！
}
```

### 实战层

**选型建议：**
- **新项目 / Kotlin 项目** → 全面使用 Flow
- **Java 项目** → 继续用 LiveData（Flow 对 Java 不友好）
- **简单 UI 绑定** → LiveData 仍然简洁（不需要 repeatOnLifecycle 样板代码）
- **复杂数据流** → Flow（combine、flatMapLatest、debounce 等）

**一次性事件处理：**
```kotlin
// LiveData 方案（有粘性问题）
class Event<T>(private val content: T) {
    private var hasBeenHandled = false
    fun getContentIfNotHandled(): T? = if (hasBeenHandled) null else { hasBeenHandled = true; content }
}

// Flow 方案（推荐）
private val _events = Channel<UiEvent>(Channel.BUFFERED)
val events = _events.receiveAsFlow()  // 每个事件只被消费一次
```

**迁移路径：**
- `LiveData` → `StateFlow`（有状态的 UI 数据）
- `SingleLiveEvent` → `SharedFlow(replay=0)` 或 `Channel`（一次性事件）
- `MediatorLiveData` → `combine` 操作符

### 延伸问题

- [[Flow冷流vs热流]]
- [[ViewModel存活原理]]
- [[Lifecycle感知组件实现]]

## 记忆锚点

LiveData = 简单但能力有限的老前辈。Flow = 功能全面的新一代。选 Flow 除非你在写 Java 或者场景极简。关键区别：Flow 需要手动处理生命周期（repeatOnLifecycle）。
