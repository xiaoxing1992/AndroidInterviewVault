---
tags: [android, 面试, kotlin, flow, S]
difficulty: S
frequency: high
---

# Flow 冷流 vs 热流

## 问题

> 请解释 Kotlin Flow 中冷流和热流的区别，SharedFlow 和 StateFlow 各自的特点和使用场景是什么？

## 核心答案

冷流（Flow）惰性触发，每个收集者独立执行上游逻辑，无收集者时不产生数据。热流（SharedFlow/StateFlow）主动发射，不依赖收集者存在，多个收集者共享同一数据源。StateFlow 是特殊的 SharedFlow，持有当前值且自动去重；SharedFlow 可配置 replay 缓存和背压策略。

## 深入解析

### 原理层

**冷流 Flow：**
```kotlin
val coldFlow = flow {
    println("开始发射")
    emit(1)
    emit(2)
}
// 每次 collect 都会重新执行 flow{} 块
coldFlow.collect { println(it) }  // 打印：开始发射 1 2
coldFlow.collect { println(it) }  // 再次打印：开始发射 1 2
```

**SharedFlow：**
```kotlin
val sharedFlow = MutableSharedFlow<Event>(
    replay = 0,              // 新订阅者收到的历史事件数
    extraBufferCapacity = 64, // 额外缓冲区
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
// 多个收集者共享同一事件流
```

**StateFlow：**
```kotlin
val stateFlow = MutableStateFlow(initialValue)
// 等价于 SharedFlow(replay=1, onBufferOverflow=DROP_OLDEST)
// 额外特性：必须有初始值、自动 distinctUntilChanged
```

**核心区别对比：**

| 特性 | Flow | SharedFlow | StateFlow |
|------|------|------------|-----------|
| 冷/热 | 冷 | 热 | 热 |
| 有无初始值 | 无 | 无 | 必须有 |
| 去重 | 无 | 无 | 自动 distinctUntilChanged |
| replay | N/A | 可配置 | 固定为 1 |
| 多收集者 | 各自独立 | 共享 | 共享 |
| 背压 | suspend | 可配置策略 | DROP_OLDEST |

**底层实现：**
- SharedFlow 内部维护一个 `SharedFlowSlot[]` 数组，每个收集者占一个 slot
- 使用 `replayCache` 环形缓冲区存储历史值
- StateFlow 的 `value` 属性通过 `StateFlowImpl` 中的 `_state: AtomicReference` 实现线程安全

### 实战层

- **替代 LiveData**：`StateFlow` + `repeatOnLifecycle` 是官方推荐的替代方案
- **一次性事件**：用 `SharedFlow(replay=0)` 处理 Toast、导航等事件，避免 StateFlow 的 replay 导致重复消费
- **多订阅者场景**：`shareIn`/`stateIn` 将冷流转为热流，避免重复请求
- **常见坑**：StateFlow 的 `distinctUntilChanged` 对 data class 使用 `equals` 比较，如果对象引用相同但内容变了不会触发更新

```kotlin
// ViewModel 中的标准用法
private val _uiState = MutableStateFlow(UiState())
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// Activity/Fragment 中收集
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state -> updateUI(state) }
    }
}
```

### 延伸问题

- [[协程原理-suspend函数与状态机]]
- [[LiveData vs Flow选型]]
- [[Channel vs Flow选型]]

## 记忆锚点

冷流像水龙头（开了才有水，各用各的），热流像广播电台（一直在播，听众共享同一频道）。StateFlow = 有记忆的广播。
