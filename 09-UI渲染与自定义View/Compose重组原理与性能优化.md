---
tags:
  - UI渲染
  - Compose
  - 重组
  - 性能优化
difficulty: S
frequency: high
---

# Compose 重组原理与性能优化

## 问题

"Compose 的重组机制是怎么工作的？Slot Table 是什么？重组作用域怎么确定？remember、derivedStateOf 怎么用？稳定性标记对性能有什么影响？"

## 核心答案（30秒版本）

Compose 通过 Slot Table 存储组合树的状态和结构，重组时只重新执行状态变化影响到的最小作用域（Composable 函数）。remember 缓存计算结果跨重组保持，derivedStateOf 将多个状态合并为派生状态减少重组次数。编译器插件为参数生成 $changed 标记，稳定类型（@Stable/@Immutable）的参数可被跳过重组（skippable），不稳定类型每次都会触发重组。

## 深入解析

### 原理层（源码级）

**Slot Table 数据结构：**

```
SlotTable = Groups[] + Slots[]

Groups: 树结构的骨架（每个 Composable 调用对应一个 Group）
  - key: Composable 的唯一标识
  - nodeCount: 子节点数量
  - dataAnchor: 指向 Slots 数组的偏移

Slots: 存储实际数据
  - State 值（mutableStateOf 的值）
  - remember 缓存的对象
  - CompositionLocal 值
  - LayoutNode 引用
```

```kotlin
// 编译器将 Composable 函数转换为：
@Composable
fun Counter(count: Int, $composer: Composer, $changed: Int) {
    $composer.startRestartGroup(key)
    
    // 检查参数是否变化
    val $dirty = $changed
    if ($changed and 0b0110 == 0) {
        $dirty = $dirty or if ($composer.changed(count)) 0b0100 else 0b0010
    }
    
    if ($dirty and 0b0011 != 0b0010 || !$composer.skipping) {
        // 参数变了或不可跳过，执行函数体
        Text("Count: $count")
    } else {
        // 参数没变，跳过重组
        $composer.skipToGroupEnd()
    }
    
    $composer.endRestartGroup()?.updateScope { composer, _ ->
        Counter(count, composer, $changed or 0b0001)
    }
}
```

**重组作用域（RestartScope）：**

```kotlin
// 每个 Composable 函数 = 一个重组作用域
// State 读取时建立订阅关系
@Composable
fun UserCard(user: User) {  // ← 重组作用域 1
    Column {                  // ← 重组作用域 2（inline，实际合并到父）
        Text(user.name)      // ← 重组作用域 3
        Text(user.email)     // ← 重组作用域 4
    }
}

// 当 user.name 变化时，只有读取了 name 的作用域被标记为 invalid
```

**State 订阅机制：**

```kotlin
// SnapshotMutableStateImpl
internal class SnapshotMutableStateImpl<T>(
    value: T,
    val policy: SnapshotMutationPolicy<T>
) : StateObject, MutableState<T> {
    
    override var value: T
        get() {
            // 读取时：记录当前 Scope 对此 State 的依赖
            Snapshot.current.readObserver?.invoke(this)
            return next.readable(this, Snapshot.current).value
        }
        set(value) {
            // 写入时：通知所有订阅的 Scope 需要重组
            val snapshot = Snapshot.current
            val old = next.writable(this, snapshot).value
            if (!policy.equivalent(old, value)) {
                next.writable(this, snapshot).value = value
                snapshot.writeObserver?.invoke(this)
            }
        }
}
```

**Recomposer 调度：**

```kotlin
// Recomposer.kt
suspend fun runRecomposeAndApplyChanges() {
    while (true) {
        // 等待有 invalid scope
        awaitWorkAvailable()
        
        // 在帧回调中执行重组
        withFrameNanos { frameTimeNs ->
            // 1. 收集所有 invalid scopes
            // 2. 按树顺序排序
            // 3. 逐个重组（调用 scope 的 block）
            // 4. 应用变更到 LayoutNode 树
            performRecompose(composition)
        }
    }
}
```

**稳定性判定规则：**

```kotlin
// 编译器自动推断稳定性：
// 稳定 = 所有公开属性都是 val + 属性类型本身稳定

// 自动稳定
data class Point(val x: Int, val y: Int) // ✓ 所有属性 val + 基本类型

// 不稳定（编译器无法确定）
data class User(val name: String, val tags: List<String>) // ✗ List 接口不稳定

// 手动标记
@Immutable // 承诺：创建后永不变化
data class Route(val path: String, val params: Map<String, String>)

@Stable // 承诺：变化时通过 Compose State 系统通知
class CounterState {
    var count by mutableStateOf(0)
}
```

### 实战层（项目经验）

**remember 正确使用：**

```kotlin
@Composable
fun ExpensiveList(items: List<Item>) {
    // ✓ 缓存计算结果，items 变化时重新计算
    val sortedItems = remember(items) {
        items.sortedBy { it.priority }
    }
    
    // ✓ 缓存对象引用，避免每次重组创建新 lambda
    val onClick = remember { { id: String -> viewModel.onItemClick(id) } }
    
    // ✗ 错误：remember 无 key，items 变化后不会更新
    val wrongSorted = remember { items.sortedBy { it.priority } }
}
```

**derivedStateOf 减少重组：**

```kotlin
@Composable
fun SearchResults(query: String, allItems: List<Item>) {
    // ✗ 每次 query 或 allItems 变化都重组整个列表
    val filtered = allItems.filter { it.name.contains(query) }
    
    // ✓ 只有过滤结果真正变化时才触发下游重组
    val filtered by remember(query, allItems) {
        derivedStateOf {
            allItems.filter { it.name.contains(query) }
        }
    }
    
    // 典型场景：滚动位置 → 是否显示 "回到顶部" 按钮
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }
}
```

**避免不必要的重组：**

```kotlin
// 1. 将读取 State 的代码下推到最小作用域
@Composable
fun AnimatedBox() {
    // ✗ 整个 Composable 每帧重组
    val offset by animateFloatAsState(targetValue)
    Box(Modifier.offset(x = offset.dp))
    
    // ✓ 只有 Modifier lambda 每帧执行，Composable 不重组
    Box(Modifier.offset { IntOffset(x = offset.roundToInt(), y = 0) })
}

// 2. 使用 key() 帮助 Compose 识别列表项
@Composable
fun ItemList(items: List<Item>) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            ItemCard(item)
        }
    }
}

// 3. 拆分 Composable 缩小重组范围
@Composable
fun UserProfile(user: User) {
    Column {
        UserAvatar(user.avatarUrl)  // avatarUrl 不变时跳过
        UserName(user.name)         // name 不变时跳过
        UserBio(user.bio)           // bio 不变时跳过
    }
}
```

**稳定性问题排查：**

```kotlin
// 使用 Compose Compiler Metrics 检查稳定性
// build.gradle.kts
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_reports")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}

// 输出示例（composables.txt）：
// restartable skippable scheme("[androidx.compose.ui.UiComposable]") fun UserCard(
//   stable name: String
//   unstable tags: List<String>  ← 导致不可跳过
// )

// 解决方案：
// 1. 使用 @Immutable 标记
// 2. 使用 kotlinx.collections.immutable (ImmutableList/PersistentList)
// 3. 包装为 @Stable 类
@Immutable
data class UserUiState(
    val name: String,
    val tags: ImmutableList<String> // kotlinx-collections-immutable
)
```

**Compose 性能工具：**

```kotlin
// 1. Layout Inspector — 查看重组次数
// 2. Composition Tracing（API 30+）
// 3. 自定义重组计数器
@Composable
fun RecompositionCounter(name: String) {
    val count = remember { mutableIntStateOf(0) }
    SideEffect { count.intValue++ }
    Log.d("Recomposition", "$name: ${count.intValue}")
}
```

### 延伸问题

- [[View绘制三大流程]] — Compose 如何绕过传统 View 的 measure/layout/draw
- [[RecyclerView缓存机制]] — LazyColumn 的 item 复用与 RecyclerView 缓存的对比
- [[属性动画原理与插值器]] — Compose 动画 API 与传统属性动画的区别
- [[屏幕适配方案对比]] — Compose 中的 dp/sp 适配方案

## 记忆锚点

Slot Table 存状态和结构，State 读取建立订阅，写入触发最小作用域重组；稳定性决定能否跳过，derivedStateOf 合并状态减少重组。
