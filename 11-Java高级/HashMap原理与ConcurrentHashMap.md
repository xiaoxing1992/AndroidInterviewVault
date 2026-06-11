---
tags: [java, hashmap, concurrenthashmap, data-structure, android]
difficulty: A
frequency: high
---

# HashMap 原理与 ConcurrentHashMap

## 问题

"HashMap 的底层数据结构是什么？put 操作的完整流程？ConcurrentHashMap 在 JDK7 和 JDK8 中的实现有什么区别？Android 中为什么推荐用 ArrayMap 替代 HashMap？"

## 核心答案（30 秒版本）

HashMap 底层是**数组 + 链表 + 红黑树**（JDK8+）。put 时先计算 hash 定位桶，冲突时链表尾插，链表长度 ≥8 且数组长度 ≥64 时转红黑树。ConcurrentHashMap 在 JDK7 用 **Segment 分段锁**，JDK8 改为 **CAS + synchronized 锁单个桶头节点**，粒度更细。Android 中 ArrayMap 用两个数组（hash 数组 + key-value 交错数组）+ 二分查找，在小数据量（<1000）时内存占用远低于 HashMap。

## 深入解析

### 原理层（源码级）

**1. HashMap 数据结构（JDK8）**

```java
// HashMap.java 核心字段
transient Node<K,V>[] table;       // 哈希桶数组
transient int size;                 // 元素数量
int threshold;                      // 扩容阈值 = capacity * loadFactor
final float loadFactor;             // 负载因子，默认 0.75

// 节点定义
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;  // 链表后继
}

// 红黑树节点
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    boolean red;
}
```

**2. put 操作完整流程**

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// 扰动函数：高16位异或低16位，减少碰撞
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1. table 为空则初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 2. 计算桶位置，为空直接放入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 3. key 相同则覆盖
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 4. 红黑树插入
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 5. 链表尾插
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表长度 >= 8，转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
    }
    // 6. 超过阈值则扩容
    if (++size > threshold)
        resize();  // 容量翻倍
    return null;
}
```

**3. 扩容机制**

```java
// resize() 核心逻辑
// 容量翻倍：oldCap << 1
// 重新分配：节点要么留在原位，要么移动到 原位置+oldCap
// 判断依据：(e.hash & oldCap) == 0 ? 原位 : 原位+oldCap
```

**4. ConcurrentHashMap JDK7 vs JDK8**

```java
// JDK7: Segment 分段锁
// 默认 16 个 Segment，每个 Segment 是一个小 HashMap + ReentrantLock
class Segment<K,V> extends ReentrantLock {
    transient volatile HashEntry<K,V>[] table;
}
// 并发度 = Segment 数量，最多 16 个线程并发写

// JDK8: CAS + synchronized
final V putVal(K key, V value, boolean onlyIfAbsent) {
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();  // CAS 初始化
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 桶为空，CAS 写入
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;
        } else {
            // 桶非空，synchronized 锁头节点
            synchronized (f) {
                // 链表或红黑树插入
            }
        }
    }
}
// 并发度 = 桶数量，锁粒度从 Segment 细化到单个桶
```

**5. Android ArrayMap 源码**

```java
// ArrayMap 内部结构
int[] mHashes;          // hash 值有序数组
Object[] mArray;        // key-value 交错存储: [key0, val0, key1, val1, ...]

// 查找：对 mHashes 二分查找
public V get(Object key) {
    final int index = indexOfKey(key);
    return index >= 0 ? (V) mArray[(index << 1) + 1] : null;
}

int indexOf(Object key, int hash) {
    final int index = Arrays.binarySearch(mHashes, 0, mSize, hash);
    // hash 冲突时线性探测
    if (index < 0) return index;
    if (key.equals(mArray[index << 1])) return index;
    // 向后/向前查找相同 hash 的 key
}
```

### 实战层（项目经验）

**1. HashMap 线程不安全的后果**

```java
// 多线程 put 可能导致：
// JDK7: 链表成环 → 死循环（头插法 + 扩容）
// JDK8: 数据丢失（尾插法解决了成环，但仍有竞态）

// 正确做法
Map<String, Object> map = new ConcurrentHashMap<>();
// 或
Map<String, Object> map = Collections.synchronizedMap(new HashMap<>());
```

**2. Android 中的选择策略**

```java
// 数据量 < 1000，key 为 int → SparseArray
SparseArray<User> users = new SparseArray<>();
users.put(userId, user);  // 避免 int 自动装箱

// 数据量 < 1000，key 为 Object → ArrayMap
ArrayMap<String, Object> config = new ArrayMap<>();

// 数据量 > 1000 或高频读写 → HashMap
HashMap<String, List<Message>> messageCache = new HashMap<>(256);

// Bundle 内部就是 ArrayMap
Bundle bundle = new Bundle();  // 内部用 ArrayMap 存储
```

**3. 内存对比**

```
存储 100 个 int→Object 映射：
HashMap:     ~3.6KB (Entry 对象 + 数组 + Integer 装箱)
SparseArray: ~1.2KB (两个数组，无装箱)
节省约 66% 内存
```

### 延伸问题

- [[JUC并发包核心组件]] — ConcurrentHashMap 中 CAS 与 AQS 的关系
- [[volatile与happens-before]] — ConcurrentHashMap 中 volatile 的应用
- [[Java泛型与类型擦除]] — Map 泛型的类型安全
- [[性能优化-内存优化]] — ArrayMap/SparseArray 在内存敏感场景的选择

## 记忆锚点

**HashMap = 数组定位 O(1) + 链表/红黑树解决冲突；ConcurrentHashMap JDK8 用 CAS+synchronized 锁单桶；Android 小数据量优先 ArrayMap/SparseArray 省内存。**
