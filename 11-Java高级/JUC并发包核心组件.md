---
tags: [java, juc, aqs, concurrent, android]
difficulty: S
frequency: medium
---

# JUC 并发包核心组件

## 问题

"AQS 的原理是什么？ReentrantLock 和 synchronized 有什么区别？CountDownLatch、Semaphore、CyclicBarrier 分别用在什么场景？在 Android 开发中这些并发工具怎么用？"

## 核心答案（30 秒版本）

AQS（AbstractQueuedSynchronizer）是 JUC 的基石，核心是一个 **volatile int state + CLH 双向队列**。获取锁时 CAS 修改 state，失败则入队自旋/阻塞。ReentrantLock 基于 AQS 实现，比 synchronized 多了**可中断、可超时、公平锁、多 Condition** 能力。CountDownLatch 用于等待多个任务完成（一次性），CyclicBarrier 用于多线程互相等待到齐后同时执行（可复用），Semaphore 控制并发数量。Android 中常用于多接口并发请求聚合、线程池限流等场景。

## 深入解析

### 原理层（源码级）

**1. AQS 核心结构**

```java
public abstract class AbstractQueuedSynchronizer {
    // 同步状态：0=未锁定，>0=已锁定（可重入次数）
    private volatile int state;

    // CLH 队列头尾指针
    private transient volatile Node head;
    private transient volatile Node tail;

    // 队列节点
    static final class Node {
        volatile int waitStatus;  // SIGNAL(-1), CANCELLED(1), CONDITION(-2)
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        Node nextWaiter;  // Condition 队列
    }
}
```

**2. AQS 获取锁流程（以 ReentrantLock 非公平锁为例）**

```java
// NonfairSync.lock()
final void lock() {
    // 1. 直接 CAS 尝试获取（非公平：插队）
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);  // 进入 AQS 标准流程
}

// AQS.acquire()
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&                    // 2. 再次尝试获取
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  // 3. 入队 + 自旋
        selfInterrupt();
}

// tryAcquire 非公平实现
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 无锁状态，CAS 获取
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        // 可重入：同一线程再次获取
        int nextc = c + acquires;
        setState(nextc);
        return true;
    }
    return false;
}

// acquireQueued：入队后自旋等待
final boolean acquireQueued(final Node node, int arg) {
    for (;;) {
        final Node p = node.predecessor();
        // 前驱是 head 时才尝试获取（保证 FIFO）
        if (p == head && tryAcquire(arg)) {
            setHead(node);
            p.next = null;  // help GC
            return false;
        }
        // 设置前驱 waitStatus=SIGNAL，然后 park 阻塞
        if (shouldParkAfterFailedAcquire(p, node) &&
            parkAndCheckInterrupt())
            interrupted = true;
    }
}
```

**3. ReentrantLock vs synchronized**

```java
// synchronized: JVM 内置，自动释放，不可中断
synchronized (lock) {
    // 临界区
}

// ReentrantLock: 手动管理，功能更强
ReentrantLock lock = new ReentrantLock(true);  // 公平锁
try {
    // 可中断获取
    lock.lockInterruptibly();
    // 超时获取
    boolean acquired = lock.tryLock(3, TimeUnit.SECONDS);
    // 多条件变量
    Condition notEmpty = lock.newCondition();
    Condition notFull = lock.newCondition();
} finally {
    lock.unlock();  // 必须手动释放
}
```

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现 | JVM 指令 monitorenter/exit | AQS + CAS |
| 释放 | 自动 | 手动 unlock |
| 可中断 | 否 | lockInterruptibly |
| 超时 | 否 | tryLock(timeout) |
| 公平性 | 非公平 | 可选公平/非公平 |
| Condition | 单一 wait/notify | 多个 Condition |
| 性能 | JDK6+ 优化后接近 | 高竞争下略优 |

**4. CountDownLatch 源码**

```java
// 基于 AQS 共享模式
public class CountDownLatch {
    private static final class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) { setState(count); }

        // await 时检查 state 是否为 0
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        // countDown 时 CAS 递减 state
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0) return false;
                int nextc = c - 1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;  // 减到 0 时唤醒所有等待线程
            }
        }
    }
}
```

**5. Semaphore 信号量**

```java
// 控制并发访问数量
Semaphore semaphore = new Semaphore(3);  // 最多 3 个线程同时访问

semaphore.acquire();  // state - 1，为 0 时阻塞
try {
    // 访问受限资源
} finally {
    semaphore.release();  // state + 1
}
```

**6. CyclicBarrier 循环屏障**

```java
// 所有线程到达屏障点后同时继续
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("所有线程到齐，开始下一阶段");
});

// 每个线程
barrier.await();  // 阻塞直到 3 个线程都调用 await
// 可复用：自动重置计数
```

### 实战层（项目经验）

**1. Android 多接口并发聚合**

```java
// 场景：首页需要同时请求 3 个接口，全部返回后刷新 UI
CountDownLatch latch = new CountDownLatch(3);
AtomicReference<UserInfo> userRef = new AtomicReference<>();
AtomicReference<List<Banner>> bannerRef = new AtomicReference<>();
AtomicReference<List<Feed>> feedRef = new AtomicReference<>();

executor.execute(() -> {
    userRef.set(api.getUserInfo());
    latch.countDown();
});
executor.execute(() -> {
    bannerRef.set(api.getBanners());
    latch.countDown();
});
executor.execute(() -> {
    feedRef.set(api.getFeeds());
    latch.countDown();
});

// 等待所有完成（带超时）
if (latch.await(5, TimeUnit.SECONDS)) {
    runOnUiThread(() -> renderPage(userRef.get(), bannerRef.get(), feedRef.get()));
} else {
    // 超时处理
}
```

**2. Semaphore 限制并发下载数**

```java
// 限制同时最多 3 个文件下载
public class DownloadManager {
    private final Semaphore semaphore = new Semaphore(3);
    private final ExecutorService executor = Executors.newCachedThreadPool();

    public void download(String url) {
        executor.execute(() -> {
            try {
                semaphore.acquire();
                doDownload(url);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                semaphore.release();
            }
        });
    }
}
```

**3. 读写锁优化缓存**

```java
// 读多写少场景：内存缓存
public class MemoryCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

    public V get(K key) {
        rwLock.readLock().lock();  // 多线程可同时读
        try {
            return cache.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void put(K key, V value) {
        rwLock.writeLock().lock();  // 写时独占
        try {
            cache.put(key, value);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

**4. Android 中的注意事项**

```java
// ❌ 不要在主线程 await/acquire
// 主线程阻塞 = ANR

// ✅ 配合协程使用
suspend fun loadHomePage(): HomeData = withContext(Dispatchers.IO) {
    val user = async { api.getUserInfo() }
    val banners = async { api.getBanners() }
    HomeData(user.await(), banners.await())  // 结构化并发
}
// Kotlin 协程在大多数场景下是更好的选择
```

### 延伸问题

- [[volatile与happens-before]] — AQS 中 volatile state 的内存语义
- [[协程调度器原理]] — Kotlin 协程如何替代 JUC 工具
- [[HashMap原理与ConcurrentHashMap]] — ConcurrentHashMap 中的 CAS 应用
- [[线程池原理与Android异步方案]] — ThreadPoolExecutor 与 AQS 的关系

## 记忆锚点

**AQS = volatile state + CAS + CLH 队列，是 Lock/Semaphore/Latch 的共同骨架；Android 实战中 CountDownLatch 聚合请求、Semaphore 限流，但优先考虑协程结构化并发。**
