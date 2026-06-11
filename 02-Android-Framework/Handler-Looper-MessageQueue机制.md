---
tags: [android, 面试, framework, handler, S]
difficulty: S
frequency: high
---

# Handler/Looper/MessageQueue 机制

## 问题

> 请详细解释 Android 的 Handler 消息机制，Looper 是如何实现无限循环而不卡死的？MessageQueue 的 epoll 机制是怎么工作的？IdleHandler 有什么用？

## 核心答案

Handler 将 Message 投递到 MessageQueue，Looper.loop() 无限循环从 MessageQueue 取消息并分发给对应 Handler 处理。MessageQueue 底层使用 Linux epoll 机制，无消息时线程阻塞在 nativePollOnce()（不消耗 CPU），有消息或超时时被唤醒。IdleHandler 在消息队列空闲时执行，适合做延迟初始化。

## 深入解析

### 原理层

**核心流程：**
```
Handler.sendMessage()
  → MessageQueue.enqueueMessage()  // 按 when 时间排序插入链表
    → nativeWake()                 // 写入 eventfd 唤醒 epoll

Looper.loop() {
  for (;;) {
    Message msg = queue.next()     // 可能阻塞
      → nativePollOnce(timeout)    // epoll_wait 阻塞
    msg.target.dispatchMessage(msg) // 分发给 Handler
    msg.recycleUnchecked()         // 回收到对象池
  }
}
```

**MessageQueue.next() 核心逻辑：**
```java
Message next() {
    for (;;) {
        nativePollOnce(ptr, nextPollTimeoutMillis); // 阻塞等待
        synchronized (this) {
            Message msg = mMessages;
            if (msg != null) {
                if (now < msg.when) {
                    nextPollTimeoutMillis = msg.when - now; // 设置超时
                } else {
                    // 取出消息返回
                    mMessages = msg.next;
                    return msg;
                }
            } else {
                nextPollTimeoutMillis = -1; // 无消息，无限等待
            }
            // 处理 IdleHandler
            if (pendingIdleHandlerCount > 0) {
                for (IdleHandler idler : mIdleHandlers) {
                    boolean keep = idler.queueIdle();
                    if (!keep) remove(idler);
                }
            }
        }
    }
}
```

**epoll 机制（native 层）：**
- `nativePollOnce` → `Looper::pollOnce` → `epoll_wait()`
- 使用 `eventfd` 作为唤醒文件描述符
- `nativeWake` → 向 eventfd 写入 1 → epoll_wait 返回 → 线程唤醒
- 为什么不卡死：epoll_wait 是阻塞系统调用，线程进入休眠状态，不占 CPU

**Message 对象池：**
```java
// Message.obtain() 从池中取，避免频繁 new
private static Message sPool;        // 链表头
private static int sPoolSize = 0;    // 当前池大小
private static final int MAX_POOL_SIZE = 50;
```

**同步屏障（SyncBarrier）：**
```java
// postSyncBarrier() 插入 target==null 的特殊 Message
// 遇到屏障时，只处理异步消息（isAsynchronous=true）
// ViewRootImpl 用它保证 UI 绘制消息优先处理
```

### 实战层

**IdleHandler 应用场景：**
```kotlin
Looper.myQueue().addIdleHandler {
    // 主线程空闲时执行，适合：
    // 1. 延迟初始化（启动优化）
    // 2. GC 触发
    // 3. 统计首帧渲染完成
    false // 返回 false 执行一次后移除，true 保持
}
```

**面试高频追问：**
- Q: 子线程能创建 Handler 吗？A: 可以，但必须先 `Looper.prepare()` + `Looper.loop()`
- Q: Handler 内存泄漏原因？A: 非静态内部类持有 Activity 引用，Message 持有 Handler 引用，MessageQueue 持有 Message
- Q: postDelayed 精确吗？A: 不精确，受前面消息处理耗时影响

**ThreadLocal 存储 Looper：**
```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<>();
// 每个线程最多一个 Looper，通过 ThreadLocal 隔离
```

### 延伸问题

- [[Activity启动流程]]
- [[卡顿优化-Choreographer与掉帧检测]]
- [[ANR分析与治理]]
- [[协程调度器原理]]

## 记忆锚点

Handler 三件套：Handler 发信、MessageQueue 排队（epoll 睡觉等信）、Looper 死循环取信派发。不卡死因为没信时 epoll 让线程睡觉不耗 CPU。
