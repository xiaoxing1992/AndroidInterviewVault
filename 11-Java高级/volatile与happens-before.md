---
tags: [java, jmm, volatile, happens-before, concurrency]
difficulty: S
frequency: high
---

# volatile 与 happens-before

## 问题

"Java 内存模型是什么？volatile 解决了什么问题？它的底层是怎么实现的？happens-before 规则有哪些？DCL 单例为什么必须加 volatile？"

## 核心答案（30 秒版本）

JMM（Java Memory Model）定义了线程与主内存之间的抽象关系：每个线程有工作内存（CPU 缓存），操作变量需从主内存拷贝。volatile 保证**可见性**（修改立即刷回主内存）和**有序性**（禁止指令重排），但**不保证原子性**。底层通过 **内存屏障（Memory Barrier）** 实现：写 volatile 前插入 StoreStore + 写后插入 StoreLoad 屏障。happens-before 是 JMM 的核心规则，定义了操作间的可见性保证。DCL 单例需要 volatile 防止对象半初始化（构造函数与引用赋值重排序）。

## 深入解析

### 原理层（源码级）

**1. JMM 抽象模型**

```
┌─────────────────────────────────────────────┐
│                  主内存 (Main Memory)         │
│         共享变量的"权威"副本                    │
└──────────┬──────────────┬───────────────────┘
           │ read/load    │ read/load
           ▼              ▼
┌──────────────┐  ┌──────────────┐
│ 线程 A 工作内存 │  │ 线程 B 工作内存 │
│ (CPU Cache)  │  │ (CPU Cache)  │
│  变量副本     │  │  变量副本     │
└──────────────┘  └──────────────┘
           │ store/write  │ store/write
           ▼              ▼
┌─────────────────────────────────────────────┐
│                  主内存 (Main Memory)         │
└─────────────────────────────────────────────┘
```

**2. 三大特性**

```java
// 可见性（Visibility）
// 线程 A 修改了变量，线程 B 能立即看到
volatile boolean running = true;

// 有序性（Ordering）
// 编译器和 CPU 不会对 volatile 读写进行重排序

// 原子性（Atomicity）
// volatile 不保证原子性！
volatile int count = 0;
count++;  // 非原子操作：read → modify → write
// 多线程下仍需 AtomicInteger 或 synchronized
```

**3. volatile 的内存屏障实现**

```java
// volatile 写操作
// 编译器生成的指令序列：
StoreStore Barrier   // 禁止上面的普通写与下面的 volatile 写重排
volatile write       // 写入 volatile 变量
StoreLoad Barrier    // 禁止上面的 volatile 写与下面的读/写重排（最重的屏障）

// volatile 读操作
volatile read        // 读取 volatile 变量
LoadLoad Barrier     // 禁止下面的普通读与上面的 volatile 读重排
LoadStore Barrier    // 禁止下面的普通写与上面的 volatile 读重排
```

**x86 架构下的实现：**

```assembly
; volatile 写 → lock 前缀指令
mov [address], value
lock addl $0, 0(%rsp)  ; lock 前缀 = 全屏障，刷新 Store Buffer

; volatile 读 → 普通 load（x86 TSO 模型天然保证 Load 不重排）
mov value, [address]
```

**4. happens-before 规则**

```java
// 规则 1: 程序顺序规则
// 同一线程中，前面的操作 happens-before 后面的操作
int a = 1;      // ①
int b = a + 1;  // ② (① happens-before ②)

// 规则 2: volatile 变量规则
// volatile 写 happens-before 后续的 volatile 读
volatile boolean flag = false;
// 线程 A
data = 42;       // ①
flag = true;     // ② volatile 写

// 线程 B
if (flag) {      // ③ volatile 读
    use(data);   // ④ 能看到 data = 42（② hb ③，传递性 ① hb ④）
}

// 规则 3: 锁规则
// unlock happens-before 后续的 lock

// 规则 4: 线程启动规则
// Thread.start() happens-before 线程中的任何操作

// 规则 5: 线程终止规则
// 线程中的任何操作 happens-before Thread.join() 返回

// 规则 6: 传递性
// A hb B, B hb C → A hb C
```

**5. DCL 单例为什么需要 volatile**

```java
public class Singleton {
    // ❌ 没有 volatile 的问题
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {           // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {   // 第二次检查
                    instance = new Singleton();  // 问题在这里！
                }
            }
        }
        return instance;
    }
}

// instance = new Singleton() 实际分三步：
// 1. 分配内存空间
// 2. 调用构造函数初始化
// 3. 将引用赋值给 instance

// CPU/编译器可能重排为 1→3→2
// 线程 A 执行到步骤 3（instance != null 但未初始化）
// 线程 B 第一次检查 instance != null，直接返回
// 线程 B 使用了未初始化的对象 → NPE 或数据错误

// ✅ 加 volatile 禁止重排序
private static volatile Singleton instance;
// volatile 写的 StoreStore 屏障确保构造函数在赋值前完成
```

**6. volatile 在 AQS 中的应用**

```java
// AbstractQueuedSynchronizer
private volatile int state;  // 同步状态

// volatile 读写 + CAS 构成无锁同步的基础
protected final int getState() { return state; }           // volatile 读
protected final void setState(int newState) { state = newState; }  // volatile 写
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
// CAS 本身包含 volatile 语义（lock cmpxchg 指令）
```

### 实战层（项目经验）

**1. Android 中 volatile 的典型使用**

```java
// 线程停止标志
public class WorkerThread extends Thread {
    private volatile boolean stopped = false;

    @Override
    public void run() {
        while (!stopped) {  // volatile 读保证可见性
            doWork();
        }
    }

    public void stopWork() {
        stopped = true;  // volatile 写立即对 run() 可见
    }
}

// HandlerThread 中的 Looper 初始化
public class HandlerThread extends Thread {
    volatile Looper mLooper;  // 跨线程可见

    @Override
    public void run() {
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();  // 通知 getLooper() 等待者
        }
        Looper.loop();
    }

    public Looper getLooper() {
        synchronized (this) {
            while (mLooper == null) {
                wait();  // 等待 Looper 初始化
            }
        }
        return mLooper;
    }
}
```

**2. volatile vs AtomicXxx**

```java
// volatile 适合：一写多读、状态标志
volatile boolean isConnected = false;

// AtomicInteger 适合：多写场景（CAS 保证原子性）
AtomicInteger requestCount = new AtomicInteger(0);
requestCount.incrementAndGet();  // 原子自增

// AtomicReference 适合：引用的原子更新
AtomicReference<Config> configRef = new AtomicReference<>(defaultConfig);
configRef.compareAndSet(oldConfig, newConfig);
```

**3. Kotlin 中的等价物**

```kotlin
// @Volatile 注解 = Java volatile
@Volatile
var running = true

// Atomic 类
val count = AtomicInteger(0)

// StateFlow 内部使用 volatile + CAS
private val _state = MutableStateFlow(initialValue)
// StateFlow 是线程安全的，适合替代 volatile 做状态管理
```

**4. 常见误区**

```java
// ❌ 误区：volatile 能保证复合操作的原子性
volatile int count = 0;
// 10 个线程各执行 1000 次 count++
// 结果 < 10000，因为 count++ 不是原子操作

// ❌ 误区：synchronized 块内不需要 volatile
// 如果变量只在 synchronized 块内访问，确实不需要
// 但 DCL 中第一次检查在 synchronized 外，所以需要 volatile

// ✅ 正确理解：volatile 是轻量级同步
// 适用场景：状态标志、一次性安全发布、独立观察
```

### 延伸问题

- [[JUC并发包核心组件]] — AQS 中 volatile state 的作用
- [[JVM内存模型与垃圾回收]] — JVM 内存区域与 JMM 的区别
- [[HashMap原理与ConcurrentHashMap]] — ConcurrentHashMap 中 volatile Node
- [[协程调度器原理]] — 协程的内存可见性保证

## 记忆锚点

**volatile = 可见性（缓存一致性）+ 有序性（内存屏障禁止重排）但不保证原子性；DCL 需要 volatile 防止构造函数与赋值重排导致读到半初始化对象。**
