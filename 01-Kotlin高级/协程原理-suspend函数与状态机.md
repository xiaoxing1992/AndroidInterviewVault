---
tags: [android, 面试, kotlin, 协程, S]
difficulty: S
frequency: high
---

# 协程原理 - suspend 函数与状态机

## 问题

> 请解释 Kotlin 协程的底层实现原理，suspend 函数编译后会变成什么样？状态机是如何工作的？

## 核心答案

suspend 函数编译后会被转换为一个带有 Continuation 参数的普通函数，函数体被编译器改写为基于 switch-case 的状态机（CPS 变换）。每个挂起点对应一个状态（label），恢复时通过 label 跳转到对应位置继续执行，从而实现非阻塞的挂起与恢复。

## 深入解析

### 原理层

编译器对 suspend 函数做了 CPS（Continuation-Passing Style）变换：

```kotlin
// 源码
suspend fun fetchData(): String {
    val token = getToken()       // 挂起点1
    val data = getData(token)    // 挂起点2
    return data
}

// 编译后伪代码
fun fetchData(continuation: Continuation<String>): Any? {
    val sm = continuation as? FetchDataSM ?: FetchDataSM(continuation)
    when (sm.label) {
        0 -> {
            sm.label = 1
            val result = getToken(sm)
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        1 -> {
            sm.token = sm.result as String
            sm.label = 2
            val result = getData(sm.token, sm)
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        2 -> {
            return sm.result as String
        }
    }
}
```

关键类与接口：
- `Continuation<T>` — 核心回调接口，包含 `resumeWith(Result<T>)` 和 `context`
- `BaseContinuationImpl` — 实现了 `resumeWith`，内部调用 `invokeSuspend`
- `CoroutineImpl` — 状态机的基类，持有 label 字段
- `COROUTINE_SUSPENDED` — 标记值，表示协程真正挂起

状态机工作流程：
1. 首次调用：label=0，执行到第一个挂起点
2. 如果子函数返回 `COROUTINE_SUSPENDED`，当前函数也返回该标记，调用栈弹出
3. 异步操作完成后，通过 `continuation.resumeWith()` 回调
4. `resumeWith` 内部调用 `invokeSuspend`，根据 label 跳转到下一个状态继续执行

### 实战层

- **调试技巧**：用 Android Studio 的 "Kotlin Bytecode" 工具查看编译后的状态机代码
- **性能认知**：协程挂起不创建新线程，只是保存状态到 Continuation 对象中，内存开销远小于线程
- **常见误区**：suspend 函数不一定会挂起 — 如果内部没有调用其他 suspend 函数或 `suspendCoroutine`，它就是同步执行的
- **堆栈恢复**：协程恢复时不会恢复调用栈，而是从状态机的对应 label 处重新进入，这就是为什么协程异常堆栈看起来不完整

### 延伸问题

- [[协程调度器原理]]
- [[协程异常处理机制]]
- [[协程作用域与结构化并发]]

## 记忆锚点

suspend = 状态机 + Continuation 回调。编译器把顺序代码拆成 switch-case，挂起点就是 label 的分界线。
