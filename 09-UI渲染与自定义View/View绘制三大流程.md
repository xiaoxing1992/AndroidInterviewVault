---
tags:
  - UI渲染
  - View
  - 绘制流程
  - 硬件加速
difficulty: S
frequency: high
---

# View 绘制三大流程

## 问题

"说一下 View 的 measure、layout、draw 三大流程，MeasureSpec 是怎么生成的？performTraversals 做了什么？硬件加速对绘制流程有什么影响？"

## 核心答案（30秒版本）

View 绘制由 ViewRootImpl.performTraversals() 触发，依次执行 performMeasure → performLayout → performDraw。measure 阶段通过 MeasureSpec（父容器约束 + 子 View LayoutParams）确定 View 大小；layout 阶段确定 View 在父容器中的位置；draw 阶段将内容绘制到 Canvas。硬件加速下 draw 阶段构建 DisplayList（RenderNode）而非直接光栅化，由 RenderThread 异步提交 GPU 渲染。

## 深入解析

### 原理层（源码级）

**performTraversals 整体流程：**

```java
// ViewRootImpl.java
private void performTraversals() {
    // 1. 预处理：窗口尺寸变化、可见性变化
    // 2. 测量
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    // 3. 布局
    performLayout(lp, mWidth, mHeight);
    // 4. 绘制
    performDraw();
}
```

**MeasureSpec 生成规则：**

MeasureSpec 是一个 32 位 int，高 2 位为 mode，低 30 位为 size：

| 父 MeasureSpec | 子 LayoutParams | 子 MeasureSpec |
|---|---|---|
| EXACTLY | 具体值 dp | EXACTLY + childSize |
| EXACTLY | MATCH_PARENT | EXACTLY + parentSize |
| EXACTLY | WRAP_CONTENT | AT_MOST + parentSize |
| AT_MOST | 具体值 dp | EXACTLY + childSize |
| AT_MOST | MATCH_PARENT | AT_MOST + parentSize |
| AT_MOST | WRAP_CONTENT | AT_MOST + parentSize |

```java
// ViewGroup.java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0, specSize - padding);
    // 根据上表规则组合
}
```

**DecorView 的顶层 MeasureSpec：**

```java
// ViewRootImpl.java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT:
            return MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            return MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        default:
            return MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
    }
}
```

**measure 流程：**

```java
// View.java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // 缓存优化：如果 MeasureSpec 未变且未强制重测，直接返回
    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;
}
```

**layout 流程：**

```java
// View.java
public void layout(int l, int t, int r, int b) {
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
    }
}
```

**draw 流程（软件渲染）：**

```java
// View.java - draw() 六步骤
public void draw(Canvas canvas) {
    // Step 1: 绘制背景 drawBackground(canvas)
    // Step 2: 保存 canvas 图层（如有必要）
    // Step 3: 绘制内容 onDraw(canvas)
    // Step 4: 绘制子 View dispatchDraw(canvas)
    // Step 5: 绘制前景、滚动条等装饰
    // Step 6: 绘制默认焦点高亮
}
```

**硬件加速下的绘制：**

```java
// ViewRootImpl.java
private boolean draw(boolean fullRedrawNeeded) {
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        // 硬件加速路径
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    } else {
        // 软件绘制路径
        drawSoftware(surface, mAttachInfo, ...);
    }
}
```

硬件加速核心机制：
1. **Recording 阶段**：View.draw() 调用的 Canvas 操作被记录为 DisplayList（RenderNode 内部）
2. **Playback 阶段**：RenderThread 遍历 DisplayList，转换为 GPU 指令
3. **局部刷新**：仅 invalidate 的 View 需要重新 record，其他 RenderNode 复用

```java
// View.java
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0 || !renderNode.hasDisplayList()) {
        RecordingCanvas canvas = renderNode.beginRecording(width, height);
        try {
            draw(canvas); // 记录绘制命令
        } finally {
            renderNode.endRecording();
        }
    }
    return renderNode;
}
```

### 实战层（项目经验）

**性能优化实践：**

1. **减少 measure 次数**：避免嵌套 RelativeLayout（会触发两次 measure），用 ConstraintLayout 替代
2. **避免过度绘制**：移除不必要的背景色，使用 `canvas.clipRect()` 裁剪
3. **利用硬件加速**：对频繁变化的 View 设置 `setLayerType(LAYER_TYPE_HARDWARE, null)` 缓存为纹理

```kotlin
// 自定义 View 正确实现 onMeasure
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    val desiredWidth = calculateDesiredWidth()
    val desiredHeight = calculateDesiredHeight()
    
    val width = resolveSize(desiredWidth, widthMeasureSpec)
    val height = resolveSize(desiredHeight, heightMeasureSpec)
    setMeasuredDimension(width, height)
}

// resolveSize 内部逻辑
fun resolveSize(desiredSize: Int, measureSpec: Int): Int {
    val specMode = MeasureSpec.getMode(measureSpec)
    val specSize = MeasureSpec.getSize(measureSpec)
    return when (specMode) {
        MeasureSpec.EXACTLY -> specSize
        MeasureSpec.AT_MOST -> minOf(desiredSize, specSize)
        else -> desiredSize // UNSPECIFIED
    }
}
```

**requestLayout vs invalidate：**

- `requestLayout()`：触发 measure + layout + draw，向上传递 PFLAG_FORCE_LAYOUT
- `invalidate()`：仅触发 draw，标记脏区域
- `postInvalidateOnAnimation()`：下一帧 VSYNC 时 invalidate，避免同一帧多次重绘

**硬件加速不支持的操作：**

```kotlin
// 以下操作在硬件加速下可能异常，需回退软件渲染
setLayerType(View.LAYER_TYPE_SOFTWARE, null)
// - Canvas.drawBitmapMesh()
// - Paint.setMaskFilter()（部分）
// - Canvas.drawVertices()
```

### 延伸问题

- [[事件分发机制]] — View 树遍历顺序与绘制顺序的关系
- [[RecyclerView缓存机制]] — RecyclerView 如何优化子 View 的 measure/layout
- [[Compose重组原理与性能优化]] — Compose 如何跳过传统 View 绘制流程
- [[屏幕适配方案对比]] — MeasureSpec 中 size 的单位与屏幕密度的关系

## 记忆锚点

performTraversals 是绘制入口，MeasureSpec = 父约束 + 子需求，硬件加速用 DisplayList 延迟渲染到 GPU。
