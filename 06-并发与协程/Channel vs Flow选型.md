---
tags: [coroutine, channel, flow, reactive, backpressure]
difficulty: A
frequency: high
---

# Channel vs Flow 选型

## 问题

"Channel 和 Flow 分别适合什么场景？什么是热流和冷流？背压怎么处理？fan-out 和 fan-in 怎么实现？"

## 核心答案（30秒版本）

Flow 是冷流，只有收集时才产生数据，天然支持背压（挂起生产者）；Channel 是热通道，数据独立于消费者产生，适合一对一或多对一的事件通信。选型原则：数据流用 Flow（StateFlow/SharedFlow），事件通信用 Channel。fan-out 用多个协程从同一 Channel 接收，fan-in 用多个协程向同一 Channel 发送。

## 深入解析

### 原理层（源码级）

**Flow 冷流本质：**

```kotlin
// Flow 只是一个接口，定义了 collect 行为
public interface Flow<out T> {
    public suspend fun collect(collector: FlowCollector<T>)
}

// flow {} builder 创建的是 SafeFlow
private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : Flow<T> {
    override suspend fun collect(collector: FlowCollector<T>) {
        // 每次 collect 都重新执行 block — 这就是"冷"的含义
        val safeCollector = SafeCollector(collector, coroutineContext)
        safeCollector.block()
    }
}

// 背压天然支持：emit 是 suspend 函数
// 当下游处理慢时，emit 会挂起，自动减慢上游速度
public interface FlowCollector<in T> {
    public suspend fun emit(value: T)  // suspend = 天然背压
}
```

**Channel 热通道本质：**

```kotlin
// Channel 内部是一个并发安全的队列
// ArrayChannel 实现（BUFFERED 策略）：
internal class ArrayChannel<E>(capacity: Int) : AbstractChannel<E>() {
    private val buffer = arrayOfNulls<Any?>(capacity)
    private var head = 0
    private var size = 0

    // send 在 buffer 满时挂起
    override suspend fun offerInternal(element: E): Any {
        if (size < capacity) {
            buffer[(head + size) % capacity] = element
            size++
            return OFFER_SUCCESS
        }
        return OFFER_FAILED  // 触发挂起
    }
}

// Channel 的几种容量策略：
Channel<Int>(Channel.RENDEZVOUS)   // 容量0，send/receive 必须会合
Channel<Int>(Channel.BUFFERED)     // 默认64
Channel<Int>(Channel.CONFLATED)    // 容量1，新值覆盖旧值
Channel<Int>(Channel.UNLIMITED)    // 无限缓冲（小心 OOM）
```

**StateFlow 源码关键：**

```kotlin
// StateFlow = 始终有值的热流，conflated
private class StateFlowImpl<T>(initialState: Any) : StateFlow<T> {
    private val _state = atomic(initialState)  // 原子状态

    override var value: T
        get() = _state.value as T
        set(value) { updateState(value) }

    // 新收集者立即收到当前值
    override suspend fun collect(collector: FlowCollector<T>) {
        val slot = allocateSlot()
        try {
            while (true) {
                val newState = slot.awaitPending()  // 挂起等待新值
                collector.emit(newState as T)
            }
        } finally {
            freeSlot(slot)
        }
    }
}

// SharedFlow = 可配置 replay 的热流
// replay=0: 新订阅者不会收到历史值
// replay=1: 类似 StateFlow 但允许相同值重复发射
```

**背压处理策略对比：**

```kotlin
// Flow 背压操作符
flow.buffer(64)           // 添加缓冲区，解耦生产消费速度
flow.conflate()           // 只保留最新值，跳过中间值
flow.collectLatest { }    // 新值来时取消上一次处理

// Channel 背压通过容量策略控制
// BufferOverflow.SUSPEND — 满了就挂起（默认）
// BufferOverflow.DROP_OLDEST — 丢弃最旧
// BufferOverflow.DROP_LATEST — 丢弃最新
```

**fan-out / fan-in 实现：**

```kotlin
// fan-out：多个消费者从同一 Channel 接收（负载均衡）
val channel = produce {
    repeat(100) { send(it) }
}
repeat(5) { workerId ->
    launch {
        for (item in channel) {  // 每个 item 只被一个 worker 处理
            process(workerId, item)
        }
    }
}

// fan-in：多个生产者向同一 Channel 发送
val channel = Channel<Event>()
launch { networkEvents.collect { channel.send(it) } }
launch { sensorEvents.collect { channel.send(it) } }
launch { userEvents.collect { channel.send(it) } }
// 单一消费者
for (event in channel) { handleEvent(event) }
```

### 实战层（项目经验）

**选型决策表：**

| 场景 | 选择 | 原因 |
|------|------|------|
| UI 状态管理 | StateFlow | 始终有值，lifecycle-aware |
| 一次性事件（Toast/导航） | Channel / SharedFlow(replay=0) | 不重复消费 |
| 数据库监听 | Flow（Room 返回） | 冷流，自动取消 |
| 搜索防抖 | Flow + debounce | 操作符丰富 |
| 多 Worker 并行处理 | Channel fan-out | 天然负载均衡 |
| WebSocket 消息 | SharedFlow | 多订阅者广播 |

**实战：一次性事件的正确处理**

```kotlin
// 方案1：Channel（推荐用于严格一次消费）
class MyViewModel : ViewModel() {
    // 使用 Channel 确保事件不丢失、不重复
    private val _events = Channel<UiEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()

    fun onButtonClick() {
        viewModelScope.launch {
            _events.send(UiEvent.NavigateTo(Screen.Detail))
        }
    }
}

// 方案2：SharedFlow（适合多观察者）
class MyViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>(
        extraBufferCapacity = 64,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events = _events.asSharedFlow()
}

// Activity/Fragment 中收集
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.NavigateTo -> navigate(event.screen)
                is UiEvent.ShowToast -> showToast(event.message)
            }
        }
    }
}
```

**实战：搜索防抖 + 取消上次请求**

```kotlin
class SearchViewModel : ViewModel() {
    private val queryFlow = MutableStateFlow("")

    val searchResults = queryFlow
        .debounce(300)                    // 防抖 300ms
        .filter { it.length >= 2 }        // 至少2个字符
        .distinctUntilChanged()           // 去重
        .flatMapLatest { query ->         // 取消上次搜索
            flow {
                emit(SearchState.Loading)
                emit(SearchState.Success(repo.search(query)))
            }.catch { emit(SearchState.Error(it)) }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), SearchState.Idle)

    fun onQueryChanged(query: String) {
        queryFlow.value = query
    }
}
```

**实战：多源数据合并**

```kotlin
// combine：所有源都有值时才发射
val homeState = combine(
    userRepo.observeUser(),
    orderRepo.observeOrders(),
    cartRepo.observeCart()
) { user, orders, cart ->
    HomeState(user, orders, cart)
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), HomeState.Loading)

// merge：任一源发射就转发
val allEvents = merge(
    networkEvents,
    localEvents,
    pushEvents
)
```

### 延伸问题

- [[协程vs线程vsRxJava对比]] — Flow 如何替代 RxJava 操作符
- [[协程作用域与结构化并发]] — Flow 收集与协程生命周期的关系
- [[线程池原理与参数调优]] — Dispatchers 底层线程池配置

## 记忆锚点

Flow 是"自来水管"——打开水龙头（collect）才有水流，天然限流；Channel 是"传送带"——物品持续放上去，多个工人可以分别取走。
