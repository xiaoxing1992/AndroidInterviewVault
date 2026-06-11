---
tags: [coroutine, thread, rxjava, concurrency, comparison]
difficulty: A
frequency: high
---

# 协程 vs 线程 vs RxJava 对比

## 问题

"协程、线程、RxJava 三者的本质区别是什么？在项目中你是怎么选型的？如果要从 RxJava 迁移到协程，你会怎么做？"

## 核心答案（30秒版本）

线程是操作系统调度的最小单位，协程是用户态的轻量级协作式调度单元，RxJava 是基于观察者模式的响应式编程框架。线程切换成本高（~1-2μs 内核态切换），协程挂起仅保存栈帧（~几十字节），RxJava 本质上还是依赖线程池但提供了声明式的异步组合能力。选型上：简单异步用协程，复杂事件流用 Flow（替代 RxJava），CPU 密集型任务直接用线程池。

## 深入解析

### 原理层（源码级）

**线程本质：**

```kotlin
// Linux 内核中线程 = 轻量级进程（LWP）
// clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, ...)
// 每个线程拥有独立栈空间（默认 1MB），共享堆内存
```

线程切换涉及：保存/恢复寄存器、刷新 TLB、内核态/用户态切换。Android 中创建一个线程的内存开销约 1MB 栈空间。

**协程本质：**

```kotlin
// 协程编译后变成状态机
// suspend fun fetchData() 编译后变成：
class FetchDataStateMachine : ContinuationImpl {
    var label = 0  // 状态标记
    var result: Any? = null

    override fun invokeSuspend(result: Result<Any?>): Any? {
        this.result = result
        when (label) {
            0 -> {
                label = 1
                // 调用挂起函数，传入 this 作为 continuation
                return networkCall(this)
            }
            1 -> {
                // 恢复执行
                return result.getOrThrow()
            }
        }
    }
}
```

协程的挂起不阻塞线程，仅保存 Continuation 对象（几十到几百字节），通过 Dispatcher 调度恢复。

**RxJava 本质：**

```java
// Observable.subscribeOn(Schedulers.io()) 底层：
// SchedulerWorker -> NewThreadWorker -> ScheduledExecutorService
// 本质还是线程池调度，只是封装了操作符链
public final class IoScheduler extends Scheduler {
    final AtomicReference<CachedWorkerPool> pool;
    // 内部维护一个可缓存的线程池
}
```

**性能对比数据：**

| 维度 | Thread | Coroutine | RxJava |
|------|--------|-----------|--------|
| 创建开销 | ~1MB 栈 | ~几十字节 Continuation | Observable 对象链 |
| 切换成本 | ~1-2μs（内核态） | ~100ns（用户态） | 依赖底层线程切换 |
| 10万并发 | OOM | 正常运行 | 取决于线程池配置 |
| 取消机制 | interrupt（不可靠） | Job.cancel（协作式） | dispose（链式取消） |
| 异常处理 | try-catch / UncaughtExceptionHandler | CoroutineExceptionHandler | onError |

### 实战层（项目经验）

**选型决策树：**

1. 简单的一次性异步操作（网络请求、数据库查询）→ 协程 `suspend fun`
2. 需要组合多个异步操作 → 协程 `async/await` 或 Flow
3. 复杂事件流（搜索防抖、合并多源数据）→ Flow（替代 RxJava）
4. CPU 密集型计算（图片处理、加密）→ `Dispatchers.Default`（底层是线程池）
5. 遗留代码中大量 RxJava → 渐进式迁移，使用 `kotlinx-coroutines-rx3` 桥接

**RxJava 迁移到协程的实战策略：**

```kotlin
// 第一步：桥接层共存
// 使用 kotlinx-coroutines-rx3 转换
fun Observable<T>.asFlow(): Flow<T>
fun Flow<T>.asObservable(): Observable<T>

// 第二步：新代码全部用协程/Flow
// Repository 层
class UserRepository {
    // 旧：fun getUser(): Single<User>
    // 新：
    suspend fun getUser(): User = withContext(Dispatchers.IO) {
        api.fetchUser()
    }
}

// 第三步：逐步替换 ViewModel 层
class UserViewModel : ViewModel() {
    // 旧：compositeDisposable.add(repo.getUser().subscribe(...))
    // 新：
    val userState = flow {
        emit(UiState.Loading)
        emit(UiState.Success(repo.getUser()))
    }.catch { emit(UiState.Error(it)) }
     .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UiState.Loading)
}
```

**迁移注意事项：**
- RxJava 的 `subscribeOn` / `observeOn` 对应协程的 `withContext(Dispatcher)`
- `CompositeDisposable` 对应结构化并发自动取消
- `flatMap` 对应 `flatMapMerge`，`concatMap` 对应 `flatMapConcat`
- 背压：RxJava Flowable 对应 Flow 的 `buffer`/`conflate`/`collectLatest`

### 延伸问题

- [[协程作用域与结构化并发]] — 协程的取消和异常传播机制
- [[Channel vs Flow选型]] — Flow 如何替代 RxJava 的各种操作符
- [[线程池原理与参数调优]] — 线程池底层原理

## 记忆锚点

线程是系统调度的"重量级马车"，协程是用户态的"轻量级自行车"，RxJava 是"马车上的自动驾驶系统"——底层还是马车，但驾驶体验更好。
