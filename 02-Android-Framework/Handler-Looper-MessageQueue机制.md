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

**nativePollOnce 详解：**

nativePollOnce 是 MessageQueue.next() 中的核心阻塞调用，底层是 Linux epoll_wait，让线程休眠等待消息，这就是 Looper.loop() 死循环不卡死的原因。

```
完整链路：
  Java 层：  MessageQueue.next() → nativePollOnce(mPtr, timeoutMillis)
  JNI 层：  android_os_MessageQueue_nativePollOnce()
            → NativeMessageQueue.pollOnce()
  Native 层：Looper::pollOnce()
            → Looper::pollInner()
              → epoll_wait(mEpollFd, events, maxEvents, timeoutMillis)
              ★ 线程阻塞在这里 ★
```

```cpp
// Looper::pollInner — 真正的 epoll_wait
int Looper::pollInner(int timeoutMillis) {
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];

    // ★ 线程在这里阻塞！
    //   timeoutMillis = 0：立即返回
    //   timeoutMillis = -1：一直等到有事件
    //   timeoutMillis > 0：等到超时或有事件
    int eventCount = epoll_wait(mEpollFd, eventItems,
                                 EPOLL_MAX_EVENTS, timeoutMillis);

    // 遍历就绪的事件
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        if (fd == mWakeEventFd) {
            // 是 wake eventfd 被写入了 → nativeWake() 唤醒的
            if (epollEvents & EPOLLIN) {
                awoken();  // read(mWakeEventFd, ...) 清除事件
            }
        } else {
            // 其他 fd 事件（如 InputChannel）
            pushResponse(events, request);
        }
    }
    return result;
}
```

```cpp
// nativeWake — 唤醒阻塞的线程
void Looper::wake() {
    uint64_t inc = 1;
    write(mWakeEventFd, &inc, sizeof(uint64_t));
    // eventfd 变为可读 → epoll_wait 返回 → 线程唤醒
}

// 谁调用 nativeWake？
//   ① MessageQueue.enqueueMessage() → 有新消息插入时
//   ② MessageQueue.quit() → 退出循环时
```

**epoll 和 eventfd 的创建：**

```cpp
// Looper 构造函数
Looper::Looper(bool allowNonCallbacks) {
    // ① 创建 epoll 实例
    mEpollFd = epoll_create1(EPOLL_CLOEXEC);

    // ② 创建 eventfd（轻量级事件通知，比 pipe 更轻量）
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);

    // ③ 将 eventfd 注册到 epoll
    struct epoll_event event;
    event.events = EPOLLIN;  // 监听"可读"事件
    event.data.fd = mWakeEventFd;
    epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, &event);
}
```

```
epoll 是什么？
  → Linux 的 I/O 多路复用机制
  → 同时监听多个 fd，有事件就绪才返回
  → 没有事件时线程休眠，不消耗 CPU

eventfd 是什么？
  → Linux 轻量级事件通知机制
  → 只需一个 fd（pipe 需要两个）
  → 只传 8 字节计数器（pipe 可以传任意数据）
  → MessageQueue 只需要"唤醒"信号，eventfd 最合适

eventfd vs pipe vs socketpair：
  eventfd：一个 fd，传 8 字节计数器，最轻量 ← MessageQueue 用这个
  pipe：两个 fd（读端/写端），传任意数据
  socketpair：两个 fd，双向通信，最重量级
```

**nativePollOnce 的三个超时值：**

```
① timeoutMillis = 0（立即返回）
  → 不阻塞，立即检查有没有消息
  → 场景：刚插入一条消息，需要立即处理

② timeoutMillis = -1（无限等待）
  → 没有消息，一直等到有新消息到来
  → 场景：MessageQueue 为空，等待 Handler 发送新消息
  → 线程休眠，不消耗 CPU ← 最常见的状态

③ timeoutMillis > 0（指定超时）
  → 等待指定毫秒后自动返回
  → 场景：最近的消息还有 N 毫秒才到执行时间
  → 等 N 毫秒后自动唤醒，检查消息是否到时间
```

```
为什么 Looper.loop() 不卡死？
  → nativePollOnce 底层是 epoll_wait
  → 无消息时：epoll_wait(-1) → 线程休眠 → CPU 占用 0%
  → 有消息时：nativeWake() 写 eventfd → epoll_wait 返回 → 处理消息
  → 和普通的 while(true) {} 完全不同
    while(true) {}     → CPU 100%，卡死
    epoll_wait()       → 线程休眠，CPU 0%，等事件唤醒
```

**Handler 消息延时实现：**

不是"定时器"，而是"时间排序 + epoll_wait 超时"。

```
核心原理：
  → Message.when = SystemClock.uptimeMillis() + delayMillis
  → MessageQueue 按 when 排序插入链表
  → nativePollOnce(timeout) 设置超时时间
  → 到时间后 epoll_wait 超时返回 → 取出消息处理
```

```java
// Handler.java
public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) delayMillis = 0;
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    // ★ when = 当前时间 + 延迟毫秒数
}
```

```java
// MessageQueue.java — 按 when 时间排序插入单链表
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        msg.when = when;

        Message prev = null;
        Message p = mMessages;  // 链表头

        if (when != 0 && when < p.when) {
            // 新消息比头节点更早 → 插到链表头
            msg.next = p;
            mMessages = msg;
        } else {
            // 遍历找到合适的位置
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
            msg.next = p;
            prev.next = msg;
        }

        if (needWake) {
            nativeWake(mPtr);  // 插到链表头时唤醒线程，重新计算超时
        }
    }
}
```

```
链表结构（按 when 时间排序）：
  mMessages → [when=100ms] → [when=200ms] → [when=500ms] → [when=1000ms] → null
    ↑
    链表头（最近要执行的消息）
```

```java
// MessageQueue.next() — 超时计算
Message next() {
    int nextPollTimeoutMillis = 0;

    for (;;) {
        nativePollOnce(mPtr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message msg = mMessages;

            if (msg != null) {
                if (now < msg.when) {
                    // ★ 消息还没到执行时间
                    // 计算还有多久到 → 设置为 epoll_wait 超时
                    nextPollTimeoutMillis = (int) Math.min(
                        msg.when - now, Integer.MAX_VALUE);
                    // 例：msg.when = now + 500ms → 超时 500ms
                } else {
                    // ★ 已到执行时间，取出消息
                    mMessages = msg.next;
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                nextPollTimeoutMillis = -1;  // 无消息，无限等待
            }
        }
    }
}
```

```
完整延时流程（以 postDelayed(runnable, 3000) 为例）：

  ① handler.postDelayed(runnable, 3000)
  ② Message.when = now + 3000 = 4000ms
  ③ MessageQueue.enqueueMessage() → 按 when 排序插入链表
  ④ next() → 取出链表头 → now=1000, msg.when=4000 → 还没到
  ⑤ nextPollTimeoutMillis = 4000 - 1000 = 3000
  ⑥ nativePollOnce(3000) → epoll_wait(timeout=3000ms) → 线程休眠 3 秒
  ⑦ 3 秒后 epoll_wait 超时返回
  ⑧ now=4000, msg.when=4000 → 到了 → 取出消息
  ⑨ msg.target.dispatchMessage(msg) → 执行 runnable
```

```
延时不精确的原因：
  ① 前面消息处理耗时 → 后面延时消息要多等
  ② 系统调度延迟 → epoll_wait 超时精度受系统影响（通常 1-10ms 误差）
  ③ GC 停顿 → STW 5-10ms

sendMessage vs sendMessageDelayed vs sendMessageAtTime：
  sendMessage(msg)
    = sendMessageDelayed(msg, 0)
      = sendMessageAtTime(msg, now)
  本质都是设置 when，然后按 when 排序插入链表
```

```java
// Message.java — 对象池是一个静态单链表
private static Message sPool;        // 链表头（静态，全局共享）
private static int sPoolSize = 0;    // 当前池大小
private static final int MAX_POOL_SIZE = 50;  // 上限 50 个
private static final Object sPoolSync = new Object();  // 全局同步锁
Message next;  // 链表指针
```

```
对象池结构：
  sPool → [Message50] → [Message49] → ... → [Message1] → null
    ↑                                              ↑
    最近回收的（链表头）                              最早回收的
    ↓                                              ↓
    obtain() 从这里取                           next 指向 null
```

**recycleUnchecked — 放入池：**

```java
// Looper.loop() 中，消息分发后调用
void recycleUnchecked() {
    // ① 清空所有字段（防止内存泄漏）
    flags = FLAG_IN_USE;
    when = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;           // ★ 清除业务数据引用
    what = 0;
    bundle = null;        // ★ 清除 Bundle 引用
    target = null;        // ★ 清除 Handler 引用（避免 Activity 泄漏）
    callback = null;      // ★ 清除 Runnable 引用
    data = null;

    // ② 头插法放入链表
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;     // 当前节点的 next → 旧头节点
            sPool = this;     // 当前节点成为新头节点
            sPoolSize++;
        }
        // 池满了：丢弃，等 GC 回收
    }
}
```

**obtain — 从池中取：**

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;         // 取头节点
            sPool = m.next;            // 头节点后移
            m.next = null;             // 断开链表
            m.flags = 0;
            sPoolSize--;
            return m;
        }
    }
    return new Message();  // 池为空才 new
}
```

```
为什么用对象池？
  → 减少 new 次数，避免频繁 GC 导致 STW 卡顿
  → 60fps 每秒可能 new 几百个 Message → GC 压力巨大
  → 对象池复用：obtain/recycle 都是 O(1) 链表操作，无堆分配，零 GC

为什么清空字段？
  → 防止内存泄漏
  → 例：msg.obj = activity，不清空 → 对象池是 static 进程级存活 → Activity 无法 GC

为什么上限 50？
  → 经验值，平衡内存（~5KB）和复用率
  → 普通 App 一帧 5-10 个 Message，50 覆盖峰值
  → 池满了丢弃等 GC，不会出错

为什么用 sPoolSync 而不是 this？
  → sPool 是静态变量，需要类锁
  → 全局唯一锁保证所有线程的 obtain/recycle 串行
```

```
完整生命周期：
  ① obtain()         → 从池中取（或 new），字段全空
  ② 设置数据         → msg.what = 1; msg.obj = "hello"
  ③ sendMessage      → MessageQueue.enqueueMessage()，按 when 插入链表
  ④ queue.next()     → 从 MessageQueue 取出
  ⑤ dispatchMessage  → Handler.handleMessage() 处理
  ⑥ recycleUnchecked → 清空字段 + 头插法放入对象池
  ⑦ 下次 obtain()    → 复用同一个对象
```

**同步屏障（SyncBarrier）：**
```java
// postSyncBarrier() 插入 target==null 的特殊 Message
// 遇到屏障时，只处理异步消息（isAsynchronous=true）
// ViewRootImpl 用它保证 UI 绘制消息优先处理
```

### 实战层

**IdleHandler 详解：**

IdleHandler = "主线程空闲时执行的回调"。MessageQueue 中没有消息要处理时，或下一条消息还没到执行时间时，调用 queueIdle() 方法。

```java
// MessageQueue.next() 中的执行位置
Message next() {
    for (;;) {
        nativePollOnce(mPtr, nextPollTimeoutMillis);
        synchronized (this) {
            // ... 取消息逻辑 ...

            // ★ IdleHandler 在这里执行
            // 条件：没有消息 或 消息还没到执行时间
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }

            for (IdleHandler idler : mPendingIdleHandlers) {
                boolean keep = idler.queueIdle();
                // ★ 返回值：
                //   true  → 保留，下次空闲时继续执行
                //   false → 执行一次后移除
            }
        }
    }
}
```

**使用方式：**

```kotlin
// 添加 IdleHandler
Looper.myQueue().addIdleHandler {
    // 主线程空闲时执行
    doSomething()
    false  // 执行一次后移除
}

// 移除 IdleHandler
Looper.myQueue().removeIdleHandler(idleHandler)
```

**典型使用场景：**

```kotlin
// 场景 1：启动优化（延迟初始化）
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        // 必须立即初始化的
        CrashReporter.init(this)
        // 延迟到主线程空闲时初始化
        Looper.myQueue().addIdleHandler {
            Analytics.init(this@MyApp)
            ImageLoader.init(this@MyApp)
            false
        }
    }
}

// 场景 2：统计首帧渲染完成
Looper.myQueue().addIdleHandler {
    reportFirstFrameRendered()  // 首帧已渲染，用户已看到页面
    false
}

// 场景 3：空闲时触发 GC
Looper.myQueue().addIdleHandler {
    System.gc()
    false
}

// 场景 4：持续监听（返回 true）
Looper.myQueue().addIdleHandler {
    if (shouldDoSomething()) {
        doSomething()
    }
    true  // 保留，每次空闲都检查
}
```

**IdleHandler vs postDelayed：**

| 维度 | IdleHandler | postDelayed |
|------|-------------|-------------|
| 触发时机 | 主线程空闲时 | 指定延迟后 |
| 时间精确 | 不精确（取决于空闲时间） | 不精确（受前面消息影响） |
| 优先级 | 最低（没有消息才执行） | 按 when 排序 |
| 适合场景 | 延迟初始化、低优先级任务 | 定时任务、延迟执行 |
| 会阻塞 UI 吗 | 会（主线程执行，但不抢占消息） | 会（主线程执行） |

```
执行位置：在 next() 中，nativePollOnce 返回后、取出消息前
  → 两个消息的间隙
  → 不会抢占正在处理的消息
  → 多个 IdleHandler 按添加顺序依次执行
```

**面试高频追问：**
- Q: 子线程能创建 Handler 吗？A: 可以，但必须先 `Looper.prepare()` + `Looper.loop()`
- Q: Handler 内存泄漏原因？A: 非静态内部类持有 Activity 引用，Message 持有 Handler 引用，MessageQueue 持有 Message
- Q: postDelayed 精确吗？A: 不精确，受前面消息处理耗时影响
- Q: IdleHandler 什么时候触发？A: MessageQueue 为空或最近消息还没到执行时间时
- Q: queueIdle 返回 true/false 什么意思？A: true 保留下次继续执行，false 执行一次后移除
- Q: IdleHandler 和 postDelayed 区别？A: IdleHandler 空闲才执行（最低优先级），postDelayed 到时间就执行

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
