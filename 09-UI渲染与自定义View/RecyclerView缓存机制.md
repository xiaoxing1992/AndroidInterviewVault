---
tags:
  - UI渲染
  - RecyclerView
  - 缓存
  - 性能优化
difficulty: S
frequency: high
---

# RecyclerView 缓存机制

## 问题

> 常见问法：
> - **RecyclerView 有几级缓存？**（最高频简化版）
> - RecyclerView 的四级缓存是怎么工作的？
> - Scrap、Cache、ViewCacheExtension、RecycledViewPool 各自的作用和区别？
> - 预取机制和 DiffUtil 是怎么优化性能的？

## 一句话回答（"几级缓存"版）

**4 级缓存**：Scrap（屏幕内临时分离，免绑定）→ CachedViews（刚滑出屏幕，默认 2 个，免绑定）→ ViewCacheExtension（自定义层，默认空实现）→ RecycledViewPool（按 viewType 分桶，每种默认 5 个，需重新 bindViewHolder）。前两级按 position 匹配复用、不重绑；后两级按 viewType 匹配、需要重绑。

## 核心答案（30秒版本）

RecyclerView 四级缓存：Scrap（当前屏幕正在 layout 的 ViewHolder，无需重新绑定）→ CachedViews（刚移出屏幕的，默认 2 个，无需绑定）→ ViewCacheExtension（开发者自定义缓存）→ RecycledViewPool（按 viewType 分桶，默认每种 5 个，需要重新 bindViewHolder）。预取机制利用 UI 线程空闲时间提前创建/绑定下一帧需要的 ViewHolder。DiffUtil 通过 Myers 差分算法计算最小更新集，避免全量 notifyDataSetChanged。

## 深入解析

### 原理层（源码级）

**缓存查找流程（Recycler.tryGetViewHolderForPositionByDeadline）：**

```java
// RecyclerView.Recycler
ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    ViewHolder holder = null;
    
    // 0. 从 changed scrap 中查找（预布局阶段）
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
    }
    
    // 1. 从 attached scrap 或 hidden views 中查找（by position）
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }
    
    // 2. 从 attached scrap 中查找（by stable id）
    if (holder == null && mAdapter.hasStableIds()) {
        holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
    }
    
    // 3. ViewCacheExtension
    if (holder == null && mViewCacheExtension != null) {
        View view = mViewCacheExtension.getViewForPositionAndType(recycler, position, type);
        if (view != null) {
            holder = getChildViewHolder(view);
        }
    }
    
    // 4. RecycledViewPool
    if (holder == null) {
        holder = getRecycledViewPool().getRecycledView(type);
        if (holder != null) {
            holder.resetInternal(); // 重置状态，需要重新绑定
        }
    }
    
    // 5. 都没有，创建新的
    if (holder == null) {
        holder = mAdapter.createViewHolder(RecyclerView.this, type);
    }
    
    // 绑定数据（Scrap 和 CachedViews 不需要绑定）
    if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        mAdapter.bindViewHolder(holder, offsetPosition);
    }
    
    return holder;
}
```

**四级缓存详解：**

| 缓存级别 | 容器 | 容量 | 是否需要绑定 | 匹配条件 |
|---|---|---|---|---|
| Scrap | mAttachedScrap / mChangedScrap | 无限制 | 否 | position |
| CachedViews | mCachedViews (ArrayList) | 默认 2 | 否 | position + viewType |
| ViewCacheExtension | 自定义 | 自定义 | 视实现 | position + viewType |
| RecycledViewPool | SparseArray<ScrapData> | 每种 type 5 个 | 是 | viewType |

**Scrap 的两种类型：**

```java
// layout 期间临时存放
final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
// 有动画变化的 item（pre-layout 到 post-layout 之间）
ArrayList<ViewHolder> mChangedScrap = null;
```

Scrap 的生命周期仅在 `onLayoutChildren()` 期间：
1. layout 开始 → 所有子 View detach 到 Scrap
2. 重新 layout → 从 Scrap 取回（position 匹配直接复用）
3. layout 结束 → Scrap 中剩余的移入 CachedViews 或 Pool

**CachedViews 的 FIFO 策略：**

```java
void recycleViewHolderInternal(ViewHolder holder) {
    if (forceRecycle || holder.isRecyclable()) {
        if (mViewCacheMax > 0 && !holder.hasAnyOfTheFlags(FLAGS)) {
            int cachedViewSize = mCachedViews.size();
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                // 满了，最老的移入 Pool
                recycleCachedViewAt(0);
            }
            mCachedViews.add(holder);
        } else {
            // 直接进 Pool
            addViewHolderToRecycledViewPool(holder, true);
        }
    }
}
```

**RecycledViewPool 结构：**

```java
public static class RecycledViewPool {
    private static final int DEFAULT_MAX_SCRAP = 5;
    
    static class ScrapData {
        final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
        int mMaxScrap = DEFAULT_MAX_SCRAP;
    }
    
    // 按 viewType 分桶
    SparseArray<ScrapData> mScrap = new SparseArray<>();
}
```

**预取机制（GapWorker）：**

```java
// GapWorker.java - 利用 VSYNC 信号后的空闲时间
final class GapWorker implements Runnable {
    @Override
    public void run() {
        // 在 UI 线程空闲时预取
        long deadlineNs = nextFrameNs; // 下一帧 VSYNC 之前完成
        prefetch(deadlineNs);
    }
    
    private void prefetch(long deadlineNs) {
        // 根据滚动方向和速度，预测下一帧需要的 item
        // 提前 createViewHolder + bindViewHolder
    }
}
```

预取触发条件：
1. RecyclerView 正在滚动（有 fling 或 touch scroll）
2. LayoutManager 实现了 `collectAdjacentPrefetchPositions()`
3. LinearLayoutManager 默认预取下一个 item

**DiffUtil Myers 算法：**

```java
// DiffUtil.java
public static DiffResult calculateDiff(Callback cb, boolean detectMoves) {
    final int oldSize = cb.getOldListSize();
    final int newSize = cb.getNewListSize();
    // Myers 差分算法：O((M+N) + D^2) 时间，O(M+N) 空间
    // D = 编辑距离（最小操作数）
    // 产出：INSERT / REMOVE / MOVE / CHANGE 操作序列
}
```

```kotlin
// 使用示例
class MyDiffCallback(
    private val oldList: List<Item>,
    private val newList: List<Item>
) : DiffUtil.Callback() {
    override fun getOldListSize() = oldList.size
    override fun getNewListSize() = newList.size
    
    override fun areItemsTheSame(oldPos: Int, newPos: Int): Boolean {
        return oldList[oldPos].id == newList[newPos].id
    }
    
    override fun areContentsTheSame(oldPos: Int, newPos: Int): Boolean {
        return oldList[oldPos] == newList[newPos]
    }
    
    override fun getChangePayload(oldPos: Int, newPos: Int): Any? {
        // 返回局部变化，避免整个 item 重新绑定
        return buildPayloadDiff(oldList[oldPos], newList[newPos])
    }
}
```

### 实战层（项目经验）

**ListAdapter + AsyncListDiffer（推荐方式）：**

```kotlin
class MyAdapter : ListAdapter<Item, MyViewHolder>(ItemDiffCallback()) {
    
    class ItemDiffCallback : DiffUtil.ItemCallback<Item>() {
        override fun areItemsTheSame(oldItem: Item, newItem: Item) = oldItem.id == newItem.id
        override fun areContentsTheSame(oldItem: Item, newItem: Item) = oldItem == newItem
    }
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyViewHolder { ... }
    
    override fun onBindViewHolder(holder: MyViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
    
    // 支持 payload 局部刷新
    override fun onBindViewHolder(holder: MyViewHolder, position: Int, payloads: List<Any>) {
        if (payloads.isEmpty()) {
            onBindViewHolder(holder, position)
        } else {
            holder.bindPartial(payloads) // 只更新变化的部分
        }
    }
}
```

**缓存优化策略：**

```kotlin
// 1. 增大 CachedViews 容量（适合 item 类型少、滚动频繁的场景）
recyclerView.setItemViewCacheSize(4)

// 2. 共享 RecycledViewPool（多个 RecyclerView 使用相同 viewType）
val sharedPool = RecyclerView.RecycledViewPool()
sharedPool.setMaxRecycledViews(VIEW_TYPE_CARD, 10)
recyclerView1.setRecycledViewPool(sharedPool)
recyclerView2.setRecycledViewPool(sharedPool) // ViewPager2 内嵌列表常用

// 3. 预创建 ViewHolder（减少滚动时的卡顿）
recyclerView.recycledViewPool.setMaxRecycledViews(0, 20)
repeat(10) {
    val holder = adapter.createViewHolder(recyclerView, 0)
    recyclerView.recycledViewPool.putRecycledView(holder)
}

// 4. setHasFixedSize(true) — 避免 item 变化触发 requestLayout
recyclerView.setHasFixedSize(true)

// 5. setHasStableIds(true) — 启用 stable id 优化动画
adapter.setHasStableIds(true)
```

**性能监控：**

```kotlin
// 自定义 RecyclerView.OnScrollListener 监控帧率
recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
        // SCROLL_STATE_SETTLING 时监控是否有掉帧
    }
})

// 通过 Choreographer 监控帧耗时
Choreographer.getInstance().postFrameCallback { frameTimeNanos ->
    val cost = System.nanoTime() - frameTimeNanos
    if (cost > 16_000_000) { // 超过 16ms
        Log.w("Perf", "Frame dropped: ${cost / 1_000_000}ms")
    }
}
```

### 延伸问题

- [[View绘制三大流程]] — RecyclerView 子 View 的 measure/layout 优化
- [[自定义LayoutManager]] — LayoutManager 如何控制缓存和回收策略
- [[Compose重组原理与性能优化]] — LazyColumn 与 RecyclerView 缓存机制的对比
- [[事件分发机制]] — RecyclerView 的 NestedScrolling 事件处理

## 记忆锚点

Scrap 是 layout 临时工，Cache 按 position 免绑定，Pool 按 type 需重绑；预取利用帧间空闲，DiffUtil 算最小更新集。
