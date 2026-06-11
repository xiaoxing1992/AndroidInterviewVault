---
tags:
  - UI渲染
  - RecyclerView
  - LayoutManager
  - 自定义View
difficulty: A
frequency: medium
---

# 自定义 LayoutManager

## 问题

"LayoutManager 的核心职责是什么？如何自定义一个 LayoutManager？onLayoutChildren 怎么实现？滚动处理和回收复用怎么做？"

## 核心答案（30秒版本）

LayoutManager 负责 RecyclerView 子 View 的测量、布局和回收策略。核心方法：generateDefaultLayoutParams() 提供默认 LayoutParams，onLayoutChildren() 执行初始布局和数据变化后的重新布局，scrollVerticallyBy()/scrollHorizontallyBy() 处理滚动并在滚动过程中回收离屏 View、填充新 View。关键是在滚动时正确调用 recycler.recycleView() 回收和 recycler.getViewForPosition() 获取，保证内存效率。

## 深入解析

### 原理层（源码级）

**LayoutManager 核心方法：**

```java
public abstract static class LayoutManager {
    // 必须实现：提供默认 LayoutParams
    public abstract LayoutParams generateDefaultLayoutParams();
    
    // 核心：布局子 View
    public void onLayoutChildren(Recycler recycler, State state) {}
    
    // 滚动支持
    public boolean canScrollVertically() { return false; }
    public boolean canScrollHorizontally() { return false; }
    public int scrollVerticallyBy(int dy, Recycler recycler, State state) { return 0; }
    public int scrollHorizontallyBy(int dx, Recycler recycler, State state) { return 0; }
    
    // 测量子 View
    public void measureChildWithMargins(View child, int widthUsed, int heightUsed) {}
    
    // 布局子 View
    public void layoutDecoratedWithMargins(View child, int left, int top, int right, int bottom) {}
    
    // 添加/移除子 View
    public void addView(View child) {}
    public void removeAndRecycleView(View child, Recycler recycler) {}
    public void detachAndScrapAttachedViews(Recycler recycler) {}
}
```

**LinearLayoutManager.onLayoutChildren 简化流程：**

```java
// LinearLayoutManager.java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // 1. 将所有已 attach 的 View 暂存到 Scrap
    detachAndScrapAttachedViews(recycler);
    
    // 2. 确定锚点（布局起始位置）
    updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
    
    // 3. 从锚点向两端填充
    // 向尾部填充
    updateLayoutStateToFillEnd(mAnchorInfo);
    fill(recycler, mLayoutState, state, false);
    
    // 向头部填充
    updateLayoutStateToFillStart(mAnchorInfo);
    fill(recycler, mLayoutState, state, false);
}

// fill 方法：循环填充直到空间用完
int fill(RecyclerView.Recycler recycler, LayoutState layoutState, State state, boolean stopOnFocusable) {
    int remainingSpace = layoutState.mAvailable;
    while (remainingSpace > 0 && layoutState.hasMore(state)) {
        // 从 Recycler 获取 ViewHolder
        View view = layoutState.next(recycler);
        // 添加到 RecyclerView
        addView(view);
        // 测量
        measureChildWithMargins(view, 0, 0);
        // 布局
        layoutDecoratedWithMargins(view, left, top, right, bottom);
        // 更新剩余空间
        remainingSpace -= consumed;
    }
    return start - remainingSpace; // 实际消耗的空间
}
```

**scrollVerticallyBy 流程：**

```java
// LinearLayoutManager.java
@Override
public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
    return scrollBy(dy, recycler, state);
}

int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
    // 1. 填充新出现的 View
    int consumed = fill(recycler, mLayoutState, state, false);
    
    // 2. 回收离屏的 View
    recycleByLayoutState(recycler, mLayoutState);
    
    // 3. 偏移所有子 View
    offsetChildrenVertical(-scrolled);
    
    return scrolled;
}
```

### 实战层（项目经验）

**自定义流式布局 LayoutManager：**

```kotlin
class FlowLayoutManager : RecyclerView.LayoutManager() {
    
    private var totalHeight = 0 // 内容总高度
    private var verticalOffset = 0 // 当前滚动偏移
    
    override fun generateDefaultLayoutParams(): RecyclerView.LayoutParams {
        return RecyclerView.LayoutParams(
            RecyclerView.LayoutParams.WRAP_CONTENT,
            RecyclerView.LayoutParams.WRAP_CONTENT
        )
    }
    
    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State) {
        if (itemCount == 0) {
            detachAndScrapAttachedViews(recycler)
            return
        }
        
        // 1. 暂存所有 View 到 Scrap
        detachAndScrapAttachedViews(recycler)
        
        // 2. 流式布局计算
        var curLineWidth = 0
        var curLineHeight = 0
        var top = 0
        var left = 0
        val parentWidth = width - paddingLeft - paddingRight
        
        for (i in 0 until itemCount) {
            val view = recycler.getViewForPosition(i)
            addView(view)
            measureChildWithMargins(view, 0, 0)
            
            val viewWidth = getDecoratedMeasuredWidth(view)
            val viewHeight = getDecoratedMeasuredHeight(view)
            
            // 换行判断
            if (curLineWidth + viewWidth > parentWidth) {
                top += curLineHeight
                left = 0
                curLineWidth = 0
                curLineHeight = 0
            }
            
            // 布局
            layoutDecoratedWithMargins(view, 
                left + paddingLeft, 
                top + paddingTop, 
                left + paddingLeft + viewWidth, 
                top + paddingTop + viewHeight
            )
            
            curLineWidth += viewWidth
            curLineHeight = maxOf(curLineHeight, viewHeight)
            left += viewWidth
        }
        
        totalHeight = top + curLineHeight + paddingTop + paddingBottom
    }
    
    override fun canScrollVertically() = true
    
    override fun scrollVerticallyBy(
        dy: Int, recycler: RecyclerView.Recycler, state: RecyclerView.State
    ): Int {
        // 边界检查
        val maxScroll = totalHeight - height
        if (maxScroll <= 0) return 0
        
        val scrolled = when {
            verticalOffset + dy < 0 -> -verticalOffset
            verticalOffset + dy > maxScroll -> maxScroll - verticalOffset
            else -> dy
        }
        
        verticalOffset += scrolled
        offsetChildrenVertical(-scrolled)
        return scrolled
    }
}
```

**带回收复用的自定义 LayoutManager（卡片堆叠效果）：**

```kotlin
class CardStackLayoutManager(
    private val maxVisibleCount: Int = 4,
    private val scaleGap: Float = 0.05f,
    private val translationYGap: Int = 14
) : RecyclerView.LayoutManager() {
    
    override fun generateDefaultLayoutParams(): RecyclerView.LayoutParams {
        return RecyclerView.LayoutParams(
            RecyclerView.LayoutParams.MATCH_PARENT,
            RecyclerView.LayoutParams.MATCH_PARENT
        )
    }
    
    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State) {
        detachAndScrapAttachedViews(recycler)
        
        if (itemCount == 0) return
        
        // 只布局可见的几张卡片
        val visibleCount = minOf(maxVisibleCount, itemCount)
        
        for (i in visibleCount - 1 downTo 0) {
            val view = recycler.getViewForPosition(i)
            addView(view)
            measureChildWithMargins(view, 0, 0)
            
            val widthSpace = width - getDecoratedMeasuredWidth(view)
            val heightSpace = height - getDecoratedMeasuredHeight(view)
            
            layoutDecoratedWithMargins(view,
                widthSpace / 2, heightSpace / 2,
                widthSpace / 2 + getDecoratedMeasuredWidth(view),
                heightSpace / 2 + getDecoratedMeasuredHeight(view)
            )
            
            // 层叠效果：越底层越小、越往下偏移
            val level = visibleCount - 1 - i
            view.scaleX = 1f - level * scaleGap
            view.scaleY = 1f - level * scaleGap
            view.translationY = (level * translationYGap).toFloat()
        }
    }
}
```

**性能优化要点：**

```kotlin
// 1. 只布局可见区域的 View
override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State) {
    detachAndScrapAttachedViews(recycler)
    
    val visibleStart = findFirstVisiblePosition()
    val visibleEnd = findLastVisiblePosition()
    
    for (i in visibleStart..visibleEnd) {
        val view = recycler.getViewForPosition(i)
        addView(view)
        measureChildWithMargins(view, 0, 0)
        layoutDecoratedWithMargins(view, ...)
    }
}

// 2. 滚动时增量回收和填充
override fun scrollVerticallyBy(dy: Int, recycler: RecyclerView.Recycler, state: RecyclerView.State): Int {
    // 回收离屏 View
    for (i in childCount - 1 downTo 0) {
        val child = getChildAt(i) ?: continue
        if (isViewOutOfBounds(child, dy)) {
            removeAndRecycleView(child, recycler)
        }
    }
    
    // 填充新进入屏幕的 View
    fillVisibleViews(recycler, dy)
    
    // 偏移
    offsetChildrenVertical(-dy)
    return dy
}

// 3. 支持预取
override fun collectAdjacentPrefetchPositions(
    dx: Int, dy: Int, state: RecyclerView.State,
    layoutPrefetchRegistry: LayoutPrefetchRegistry
) {
    // 告诉 RecyclerView 下一帧可能需要的 position
    val nextPosition = if (dy > 0) findLastVisiblePosition() + 1 else findFirstVisiblePosition() - 1
    if (nextPosition in 0 until state.itemCount) {
        layoutPrefetchRegistry.addPosition(nextPosition, abs(dy))
    }
}
```

**SnapHelper 配合自定义 LayoutManager：**

```kotlin
// 让滚动停止时对齐到某个 item
val snapHelper = PagerSnapHelper() // 或 LinearSnapHelper
snapHelper.attachToRecyclerView(recyclerView)

// 自定义 SnapHelper
class CardSnapHelper : SnapHelper() {
    override fun calculateDistanceToFinalSnap(
        layoutManager: RecyclerView.LayoutManager, targetView: View
    ): IntArray {
        val distances = IntArray(2)
        distances[0] = targetView.left - layoutManager.paddingLeft
        distances[1] = 0
        return distances
    }
    
    override fun findSnapView(layoutManager: RecyclerView.LayoutManager): View? {
        return findCenterView(layoutManager)
    }
}
```

### 延伸问题

- [[RecyclerView缓存机制]] — LayoutManager 如何与 Recycler 四级缓存协作
- [[View绘制三大流程]] — LayoutManager 中 measureChildWithMargins 的 MeasureSpec 生成
- [[事件分发机制]] — LayoutManager 中的滚动事件与 NestedScrolling
- [[Compose重组原理与性能优化]] — Compose LazyLayout 与自定义 LayoutManager 的对比

## 记忆锚点

onLayoutChildren 做初始布局，scrollBy 做增量填充+回收+偏移；核心是只布局可见区域，滚动时增量更新。
