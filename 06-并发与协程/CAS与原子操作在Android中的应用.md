---
tags: [cas, atomic, lock-free, concurrency, performance]
difficulty: A
frequency: medium
---

# CAS 与原子操作在 Android 中的应用

## 问题

"CAS 的原理是什么？ABA 问题怎么解决？AtomicInteger 底层怎么实现的？在 Android 开发中有哪些实际应用场景？和加锁相比性能如何？"

## 核心答案（30秒版本）

CAS（Compare-And-Swap）是一条 CPU 原子指令（x86 的 CMPXCHG），比较内存值与期望值相等则更新为新值，否则失败重试。AtomicInteger 通过 Unsafe.compareAndSwapInt 实现无锁自增。ABA 问题用 AtomicStampedReference（版本号）解决。Android 中 CAS 广泛用于 Handler 消息标记、协程状态机、View 属性动画、引用计数等场景。低竞争下 CAS 性能远优于锁，高竞争下自旋浪费 CPU。

## 深入解析

### 原理层（源码级）

**CAS 硬件支持：**

```
// x86 架构：CMPXCHG 指令（带 LOCK 前缀保证多核原子性）
LOCK CMPXCHG [destination], source
// 如果 EAX == [destination]，则 [destination] = source，ZF=1
// 否则 EAX = [destination]，ZF=0

// ARM 架构：LL/SC（Load-Linked / Store-Conditional）
LDREX R1, [R0]     // Load-Exclusive
CMP R1, R2         // 比较
STREXEQ R3, R4, [R0]  // Store-Exclusive（条件存储）
// 如果 R0 地址在 LDREX 后未被其他核修改，则存储成功
```

**AtomicInteger 源码：**

```java
public class AtomicInteger extends Number {
    // Unsafe 提供 CAS 原语
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(
        AtomicInteger.class.getDeclaredField("value"));

    private volatile int value;  // volatile 保证可见性

    public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
    }

    // Unsafe.getAndAddInt 实现（JDK 8+）：
    // public final int getAndAddInt(Object o, long offset, int delta) {
    //     int v;
    //     do {
    //         v = getIntVolatile(o, offset);  // 读当前值
    //     } while (!compareAndSwapInt(o, offset, v, v + delta));  // CAS 循环
    //     return v;
    // }
}
```

**ABA 问题详解：**

```
线程1：读到 A，准备 CAS(A → C)
线程2：A → B → A（值变回 A）
线程1：CAS 成功（但中间状态被忽略了！）

// 场景：无锁栈的 pop
// 栈：A → B → C
// 线程1：准备 pop A，next = B
// 线程2：pop A, pop B, push A（栈变成 A → C）
// 线程1：CAS(head, A, B) 成功，但 B 已经不在栈中了！→ 数据丢失
```

**ABA 解决方案：**

```java
// 方案1：AtomicStampedReference（版本号）
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(100, 0);

int[] stampHolder = new int[1];
Integer current = ref.get(stampHolder);
int stamp = stampHolder[0];

// CAS 时同时比较值和版本号
ref.compareAndSet(current, newValue, stamp, stamp + 1);

// 方案2：AtomicMarkableReference（布尔标记）
AtomicMarkableReference<Node> ref = new AtomicMarkableReference<>(node, false);
```

**LongAdder vs AtomicLong（高竞争优化）：**

```java
// AtomicLong：所有线程竞争同一个 value
// 高竞争下大量 CAS 失败重试

// LongAdder：分段思想（类似 ConcurrentHashMap）
// 内部维护 Cell[] 数组，每个线程 hash 到不同 Cell
// sum() 时汇总所有 Cell
public class LongAdder extends Striped64 {
    // base + cells[0].value + cells[1].value + ...
    public void add(long x) {
        Cell[] cs; long b, v; int m; Cell c;
        if ((cs = cells) != null || !casBase(b = base, b + x)) {
            // base CAS 失败，分散到 Cell
            int index = getProbe() & m;
            if (!cs[index].cas(v = cs[index].value, v + x)) {
                longAccumulate(x, null, uncontended);
            }
        }
    }
}

// 性能对比（8线程并发自增 1000万次）：
// AtomicLong: ~800ms
// LongAdder:  ~150ms（5x 提升）
// 但 LongAdder 的 sum() 不是精确快照
```

### 实战层（项目经验）

**Android 中的 CAS 应用场景：**

**1. 协程状态机（核心使用）：**

```kotlin
// AbstractCoroutine 中的状态转换
// state: New → Active → Completing → Completed/Cancelled
// 使用 AtomicReference + CAS 保证状态转换原子性
private val _state = atomic<Any?>(if (active) ACTIVE else NEW)

private fun makeCompleting(proposedUpdate: Any?): Any? {
    loopOnState { state ->
        val finalState = tryMakeCompleting(state, proposedUpdate)
        if (finalState !== COMPLETING_RETRY) return finalState
    }
}
```

**2. Handler 消息标记：**

```java
// Message.java 中的 flags 使用原子操作
// FLAG_IN_USE = 1 << 0
// FLAG_ASYNCHRONOUS = 1 << 1
/*package*/ int flags;

// 标记消息正在使用（回收时检查）
void markInUse() {
    flags |= FLAG_IN_USE;
}
```

**3. View 属性动画计数器：**

```kotlin
// ObjectAnimator 内部使用 AtomicInteger 管理动画 ID
private static final AtomicInteger sAnimationID = new AtomicInteger(0);
```

**4. 无锁计数器（埋点/统计）：**

```kotlin
class EventCounter {
    private val counts = ConcurrentHashMap<String, LongAdder>()

    fun record(event: String) {
        counts.getOrPut(event) { LongAdder() }.increment()
    }

    fun snapshot(): Map<String, Long> =
        counts.mapValues { it.value.sum() }
}
```

**5. 无锁单例初始化：**

```kotlin
class LazyResource<T>(private val initializer: () -> T) {
    private val ref = AtomicReference<T>(null)

    fun get(): T {
        ref.get()?.let { return it }
        val newValue = initializer()
        // CAS 保证只有一个线程的初始化结果被采用
        return if (ref.compareAndSet(null, newValue)) {
            newValue
        } else {
            ref.get()!!  // 其他线程已初始化
        }
    }
}
```

**6. 无锁队列（生产者-消费者）：**

```kotlin
// 基于 CAS 的无锁 MPSC（多生产者单消费者）队列
class MpscQueue<T> {
    private val head = AtomicReference<Node<T>>(Node(null))
    private var tail: Node<T> = head.get()

    fun offer(item: T) {
        val newNode = Node(item)
        while (true) {
            val currentTail = head.get()
            if (head.compareAndSet(currentTail, newNode)) {
                currentTail.next = newNode
                return
            }
        }
    }
}
```

**7. 协程 Flow 中的原子状态：**

```kotlin
// StateFlow 内部使用 atomic 保证状态更新原子性
class MutableStateFlow<T>(initialValue: T) {
    private val _state = atomic(initialValue)

    override var value: T
        get() = _state.value
        set(value) {
            // CAS 循环更新
            while (true) {
                val cur = _state.value
                if (cur == value) return  // 相同值不更新
                if (_state.compareAndSet(cur, value)) {
                    // 通知收集者
                    return
                }
            }
        }
}
```

**CAS vs 锁的选型：**

| 场景 | 推荐 | 原因 |
|------|------|------|
| 简单计数器 | AtomicInteger/LongAdder | 无锁，性能好 |
| 状态标记（boolean） | AtomicBoolean | 一条指令 |
| 引用更新 | AtomicReference | 无锁 CAS |
| 复合操作（check-then-act） | 锁 | CAS 无法保证复合原子性 |
| 高竞争 + 长临界区 | 锁 | CAS 自旋浪费 CPU |
| 高竞争 + 短操作 | LongAdder | 分段减少竞争 |

**性能测试模板：**

```kotlin
fun benchmarkConcurrency() {
    val iterations = 1_000_000
    val threads = 8

    // synchronized
    var syncCount = 0
    val syncLock = Any()
    val syncTime = measureTimeMillis {
        runBlocking(Dispatchers.Default) {
            repeat(threads) {
                launch {
                    repeat(iterations / threads) {
                        synchronized(syncLock) { syncCount++ }
                    }
                }
            }
        }
    }

    // AtomicInteger
    val atomicCount = AtomicInteger(0)
    val atomicTime = measureTimeMillis {
        runBlocking(Dispatchers.Default) {
            repeat(threads) {
                launch {
                    repeat(iterations / threads) {
                        atomicCount.incrementAndGet()
                    }
                }
            }
        }
    }

    Log.d("Bench", "synchronized: ${syncTime}ms, atomic: ${atomicTime}ms")
}
```

### 延伸问题

- [[synchronized vs ReentrantLock vs Mutex]] — 锁的实现原理对比
- [[线程池原理与参数调优]] — 线程池内部使用 CAS 管理 Worker 状态
- [[死锁检测与避免]] — 无锁编程避免死锁

## 记忆锚点

CAS 就像"乐观抢票"——先看座位空着（Compare），直接坐下（Swap），如果被人抢了就重新找座位（自旋重试）。锁是"悲观排队"——不管有没有人，先排队再说。
