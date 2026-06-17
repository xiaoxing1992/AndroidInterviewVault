---
tags: [android, 面试, framework, activity, view, rendering, window, surface, S]
difficulty: S
frequency: high
---

# Activity 显示原理

## 问题

> Activity 从 onCreate 到用户看到画面经历了哪些步骤？PhoneWindow、DecorView、ViewRootImpl、Surface、SurfaceFlinger 分别起什么作用？硬件加速和软件渲染有什么区别？

## 核心答案

Activity 显示链路：onCreate 中 setContentView 创建 DecorView（View 树根）→ onResume 后 ViewRootImpl 创建并调用 WMS.addWindow 注册窗口获取 Surface → VSYNC 信号到来触发 performTraversals（measure→layout→draw）→ 绘制内容到 Surface 的 Buffer → BufferQueue 传递给 SurfaceFlinger 合成 → HWC 送显。关键组件：PhoneWindow（Window 唯一实现）、DecorView（View 树根）、ViewRootImpl（连接 View 系统和 WMS 的桥梁）、Surface（绘图画布）、BufferQueue（App 与 SurfaceFlinger 的共享内存通道）。

## 深入解析

### 原理层（源码级）

**完整层次图：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                          应用层 (App Process)                        │
│                                                                     │
│  Activity → PhoneWindow → DecorView → View 树                       │
│                                          ↓                          │
│  ViewRootImpl ← 桥梁                                                │
│  ├── performTraversals() → measure / layout / draw                  │
│  ├── Surface                           ← 绘图画布                   │
│  └── IWindowSession                    ← 与 WMS 的 Binder 通道      │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                       框架层 (system_server)                         │
│                                                                     │
│  WMS (WindowManagerService)                                         │
│  ├── WindowState            ← 每个窗口的数据结构                     │
│  ├── Surface 分配            ← 管理每个窗口的绘图表面                 │
│  ├── Z-order 管理           ← 窗口前后层级                          │
│  └── 输入事件分发            ← 触摸事件发给哪个窗口                   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                      Native 层                                      │
│                                                                     │
│  Surface → BufferQueue → SurfaceFlinger → HWC → Display             │
│            (mmap 共享)   (合成所有 Layer)   (硬件合成)  (屏幕)        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

#### 阶段 1：PhoneWindow 创建（Activity.attach）

```java
// Activity.java
final void attach(Context context, ActivityThread aThread, ...) {
    // 创建 PhoneWindow（Window 的唯一实现）
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setCallback(this);  // Activity 作为 Window 的回调

    // 设置 WindowManager
    mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(), ...);
}
```

```
Activity
  └── PhoneWindow（Window 的唯一实现）
        ├── DecorView（View 树的根）
        └── WindowManagerImpl
```

---

#### 阶段 2：setContentView → DecorView 创建

```java
// PhoneWindow.java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();  // 创建 DecorView
    }
    mLayoutInflater.inflate(layoutResID, mContentParent);
    // 用户的布局被添加到 mContentParent 中
}

private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor();  // new DecorView(...)
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
        // 根据主题选择布局模板
    }
}
```

```
DecorView（FrameLayout）
  ├── LinearLayout（系统布局模板）
  │     ├── 标题栏区域（ActionBar / Toolbar）
  │     └── android.R.id.content（FrameLayout）
  │           └── 用户 setContentView 的布局 ← mContentParent
  │                 ├── Button
  │                 ├── TextView
  │                 └── ...

注意：此时只是创建了 View 树，还没有测量、布局、绘制，也没有上屏
```

---

#### 阶段 3：onResume → ViewRootImpl 创建 → WMS 注册

```java
// ActivityThread.java
public void handleResumeActivity(...) {
    // 1. 调用 Activity.onResume()
    final ActivityClientRecord r = performResumeActivity(...);

    // 2. 把 DecorView 添加到 WindowManager
    ViewManager wm = a.getWindowManager();
    View decor = r.activity.getWindow().getDecorView();
    WindowManager.LayoutParams l = r.window.getAttributes();
    wm.addView(decor, l);
    // → WindowManagerImpl.addView()
    // → WindowManagerGlobal.addView()
}
```

```java
// WindowManagerGlobal.java
public void addView(View view, ViewGroup.LayoutParams params, ...) {
    // 创建 ViewRootImpl（核心！连接 View 和 WMS 的桥梁）
    root = new ViewRootImpl(view.getContext(), display);

    // 关联 ViewRootImpl → DecorView
    mViews.add(view);       // DecorView
    mRoots.add(root);       // ViewRootImpl
    mParams.add(wparams);   // LayoutParams

    // 设置 View 树的根
    root.setView(view, wparams, panelParentView, userId);
}
```

```java
// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, ...) {
    mView = view;  // DecorView

    // 1. 开启硬件加速
    enableHardwareAcceleration(attrs);

    // 2. 先 requestLayout — 预约下一帧 VSYNC 时执行 performTraversals
    requestLayout();
    // → scheduleTraversals()
    // → Choreographer.postCallback(TRAVERSAL, mTraversalRunnable)
    // → 注册 VSYNC 监听（但不立即执行 performTraversals）

    // 3. 再向 WMS 注册窗口（同步 Binder 调用）
    res = mWindowSession.addToDisplayAsUser(mWindow, ...);
    // → WMS 创建 WindowState、分配 Surface
    // → 返回后窗口已注册，输入事件通道建立

    // 4. 创建 InputEventReceiver（接收输入事件）
    mInputEventReceiver = new WindowInputEventReceiver(inputChannel, ...);

    // 5. 后续 VSYNC 到来时执行 performTraversals
    // → measure → layout → draw → 上屏
}
```

源码注释原文：

```java
// Schedule the first layout -before- adding to the window
// manager, to make sure we do the relayout before receiving
// any other events from the system.
requestLayout();
```

**requestLayout() 详解：**

```java
// ViewRootImpl.java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();  // 检查是否在主线程
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;

        // 1. 设置同步屏障（优先处理异步消息，即 VSYNC 回调）
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();

        // 2. 向 Choreographer 注册 TRAVERSAL 回调
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL,
            mTraversalRunnable,    // ← VSYNC 到来时执行
            null);

        // 3. 通知 InputManager
        notifyRendererOfFramePending();
    }
}

// mTraversalRunnable 就是：
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        performTraversals();  // 真正执行绘制
    }
}
```

requestLayout 做了什么：① 标记 mLayoutRequested=true ② 插入同步屏障 ③ Choreographer 注册 VSYNC 回调 ④ 不执行 performTraversals，只是"预约"。

**addToDisplayAsUser() 详解：**

```java
// mWindowSession 是 IWindowSession 的 Binder 接口，实现在 WMS 侧的 Session.java
// mWindow 是 W extends IWindow.Stub，是 ViewRootImpl 暴露给 WMS 的 Binder 接口

res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes, ...);
```

WMS 侧收到后：

```java
// Session.java (system_server 进程)
@Override
public int addToDisplayAsUser(IWindow window, ...) {
    return mService.addWindow(this, window, ...);
}

// WindowManagerService.java
public int addWindow(Session session, IWindow client, ...) {
    // 1. 权限检查
    // 2. 创建 WindowState（窗口在 WMS 中的数据结构）
    WindowState win = new WindowState(this, session, client, token, ...);
    // 3. 计算窗口层级 Z-order
    // 4. 建立 InputChannel（输入事件通道）
    // 5. 返回结果给 App
    return WindowManagerGlobal.ADD_OKAY;
}
```

addToDisplayAsUser 返回后 App 侧继续：

```java
// 1. 创建输入事件接收器
mInputEventReceiver = new WindowInputEventReceiver(inputChannel, Looper.myLooper());

// 2. 建立输入事件处理管线（责任链模式）
mFirstInputStage = nativePreImeStage;
// 处理顺序：NativePreIme → ViewPreIme → Ime → EarlyPostIme → NativePostIme → ViewPostIme → Synthetic

// 3. 设置 DecorView 的 parent 为 ViewRootImpl
view.assignParent(this);
// 这就是为什么 ViewRootImpl 是 View 树的根节点
```

**先 requestLayout 再 addToDisplay 的原因：**

```
requestLayout()                    addToDisplayAsUser()
──────────────                     ────────────────────
App 进程内操作                      App → WMS（跨进程 Binder）

做了什么：                          做了什么：
① 标记 mLayoutRequested=true       ① WMS 创建 WindowState
② 插入同步屏障                      ② WMS 分配 Surface（初步）
③ Choreographer 注册 VSYNC 回调     ③ WMS 建立 InputChannel
④ 不执行 performTraversals          ④ 同步阻塞等 WMS 返回
```

```
假设反过来（先 addToDisplay 再 requestLayout）：
  WMS 返回后，输入事件可能立刻到来
  但此时还没预约 VSYNC → performTraversals 还没调度
  如果 VSYNC 在 requestLayout 之前到达 → 错过这一帧 → 延迟一帧显示

正确顺序（先 requestLayout 再 addToDisplay）：
  VSYNC 回调已经注册好
  WMS 返回后，无论 VSYNC 何时到来都能立刻响应
  不会因为"还没预约"而延迟一帧
```

```
完整时序：
  setView() {
      requestLayout()                    ← 预约 VSYNC（不执行）
        → postSyncBarrier()
        → Choreographer.postCallback(TRAVERSAL)

      addToDisplayAsUser()               ← 同步 Binder 调用
        → WMS: 创建 WindowState
        → WMS: 分配 Surface
        → WMS: 建立 InputChannel
        → 返回

      InputEventReceiver 创建            ← 接收输入事件
      DecorView.parent = ViewRootImpl

      setView() 结束
  }

  // 主线程 Looper 继续运行，等待 VSYNC

  VSYNC 信号到达
    → Choreographer.doFrame()
      → mTraversalRunnable.run()
        → doTraversal()
          → removeSyncBarrier()
          → performTraversals()          ← 此时才真正执行
            → relayoutWindow()           ← 向 WMS 请求真正的 Surface
            → performMeasure()
            → performLayout()
            → performDraw()              ← 绘制到 Surface
```

---

#### 阶段 4：VSYNC 信号 → performTraversals

```
VSYNC 信号到达
  → Choreographer.doFrame()
    → mTraversalRunnable.run()
      → ViewRootImpl.doTraversal()
        → performTraversals()
          → performMeasure()
          → performLayout()
          → performDraw()
```

```java
// ViewRootImpl.java
private void performTraversals() {
    // ① 预处理：计算顶层 MeasureSpec
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

    // ② 测量：确定 View 树中每个 View 的大小
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

    // ③ 布局：确定 View 树中每个 View 的位置
    performLayout(lp, mWidth, mHeight);

    // ④ 绘制：将内容绘制到 Surface 的 Canvas 上
    performDraw();
}
```

**performMeasure → DecorView.measure → 递归测量所有子 View**

```java
// View.java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // 缓存优化：MeasureSpec 未变且未强制重测，直接返回
    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
}
```

**performLayout → DecorView.layout → 递归布局所有子 View**

```java
// View.java
public void layout(int l, int t, int r, int b) {
    boolean changed = setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) != 0) {
        onLayout(changed, l, t, r, b);
    }
}
```

**performDraw → 绘制到 Surface**

```java
// ViewRootImpl.java
private void performDraw() {
    boolean canUseAsync = draw(fullRedrawNeeded);
}

private boolean draw(boolean fullRedrawNeeded) {
    if (mAttachInfo.mThreadedRenderer != null &&
            mAttachInfo.mThreadedRenderer.isEnabled()) {
        // 硬件加速路径（默认）
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    } else {
        // 软件渲染路径
        drawSoftware(surface, mAttachInfo, ...);
    }
}

// 软件渲染
private boolean drawSoftware(Surface surface, ...) {
    final Canvas canvas = mSurface.lockCanvas(dirty);  // 锁定 Buffer
    mView.draw(canvas);                                 // 绘制 View 树
    surface.unlockCanvasAndPost(canvas);                // 提交 Buffer 到 BufferQueue
}
```

---

#### 阶段 5：View.draw() 的六个步骤

```java
// View.java
public void draw(Canvas canvas) {
    // Step 1: 绘制背景
    drawBackground(canvas);

    // Step 2: 保存 canvas 图层（如有 fading edge）
    // ...

    // Step 3: 绘制自身内容
    onDraw(canvas);

    // Step 4: 绘制子 View
    dispatchDraw(canvas);

    // Step 5: 绘制前景、滚动条、装饰
    onDrawForeground(canvas);

    // Step 6: 绘制默认焦点高亮
    drawDefaultFocusHighlight(canvas);
}
```

```
递归绘制顺序（DFS）：
  DecorView.draw()
    → DecorView.onDraw()
    → DecorView.dispatchDraw()
      → LinearLayout.draw()
        → LinearLayout.dispatchDraw()
          → TextView.draw()
            → TextView.onDraw()  ← 文字渲染
          → Button.draw()
            → Button.onDraw()    ← 按钮渲染
```

---

#### 阶段 6：Surface → BufferQueue → SurfaceFlinger

```
App 进程 (Producer)         BufferQueue          SurfaceFlinger (Consumer)
───────────────────         ───────────          ────────────────────────
lockCanvas() / dequeueBuffer()
  → 获取一块空闲 Buffer
  → Canvas 绘制内容到 Buffer
unlockCanvas() / queueBuffer()
  → 提交已填充的 Buffer
                               → Buffer 进入队列
                                                    acquireBuffer()
                                                      → 获取 Buffer
                                                      → 合成所有 Window 的 Layer
                                                      → HWC (Hardware Composer)
                                                      → Display HAL
                                                      → 用户看到画面
                                                    releaseBuffer()
                                                      → 回收 Buffer
```

```
BufferQueue 机制：
  通常有 3 个 Buffer（Triple Buffering）
  → App 绘制 Buffer A
  → SurfaceFlinger 合成 Buffer B
  → Display 显示 Buffer C
  → 三者并行，减少卡顿

通信方式：mmap 共享内存（零拷贝，不经过 Binder）
```

---

### 硬件加速 vs 软件渲染

```
软件渲染：
  View.draw() → Canvas 操作直接光栅化到 Bitmap Buffer → 提交
  → 主线程完成所有绘制
  → 每帧都要重新绘制整个脏区域

硬件加速（默认开启 API 14+）：
  View.draw() → Canvas 操作记录为 DisplayList (RenderNode)
  → 主线程 Recording 阶段结束
  → RenderThread 遍历 DisplayList → 转换为 GPU 指令 → GPU 渲染
  → 局部更新：只有 invalidate 的 View 需要重新 Recording
  → 其他 RenderNode 直接复用
```

```java
// 硬件加速下 View 绘制
// View.java
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    if (!renderNode.hasDisplayList()) {
        RecordingCanvas canvas = renderNode.beginRecording(width, height);
        try {
            draw(canvas);  // 记录绘制命令到 DisplayList，不直接光栅化
        } finally {
            renderNode.endRecording();
        }
    }
    return renderNode;
}
```

```
硬件加速绘制流程：
  主线程                        RenderThread                  GPU
  ──────                       ─────────────                  ───
  performTraversals()
    → performDraw()
      → updateDisplayListIfDirty()
        → 构建/更新 DisplayList
      → ThreadedRenderer.draw()
        → syncFrameState()
        → postAndWait()  ──→  遍历 DisplayList
                              → 转换为 GL/Vulkan 指令
                              → eglSwapBuffers()  ──→  渲染到 Buffer
                                                     → 提交到 BufferQueue
```

---

### WMS 核心职责

```
WMS (WindowManagerService) 运行在 system_server 进程：

1. 窗口管理：addWindow / removeWindow / relayoutWindow
2. 布局计算：确定每个窗口的位置和大小
3. Z-order 管理：决定窗口的前后层级
4. 动画管理：窗口切换动画
5. 输入事件分发：确定触摸事件发给哪个窗口

Window 类型与层级（Z-order）：
  TYPE_APPLICATION (1-99)        — 普通 Activity 窗口
  TYPE_APPLICATION_OVERLAY (2038) — 悬浮窗
  TYPE_STATUS_BAR (2000)         — 状态栏
  TYPE_TOAST (2005)              — Toast
  TYPE_SYSTEM_ALERT (2003)       — 系统弹窗
  // Z-order 越大越靠前
```

**ViewRootImpl 的桥梁作用：**

```java
// ViewRootImpl 连接 View 系统和 WMS
class ViewRootImpl {
    final Surface mSurface = new Surface();  // 绘图表面
    final W mWindow;                          // Binder，WMS 回调用

    void setView(View view, ...) {
        // 1. 向 WMS 添加窗口
        mWindowSession.addToDisplayAsUser(mWindow, ...);
        // 2. 请求布局
        requestLayout();
    }

    void performTraversals() {
        // measure → layout → draw 三大流程
        performMeasure();
        performLayout();
        performDraw();  // 最终绘制到 Surface
    }
}
```

---

### 实战层（项目经验）

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| ViewRootImpl 是什么时候创建的？ | handleResumeActivity 中，wm.addView(decor) 时创建 |
| DecorView 是什么时候创建的？ | setContentView → installDecor → generateDecor() |
| 为什么 onResume 之后 View 才可见？ | ViewRootImpl 在 onResume 后才创建，performTraversals 才执行 |
| performTraversals 什么时候触发？ | VSYNC 信号到来时，通过 Choreographer 调度 |
| Surface 是谁创建的？ | WMS 分配，App 通过 ViewRootImpl 持有引用 |
| View.invalidate 和 requestLayout 区别？ | invalidate 只触发 draw，requestLayout 触发 measure+layout+draw |
| 硬件加速和软件渲染区别？ | 硬件加速用 DisplayList + GPU，软件用 Canvas 直接光栅化 |
| 为什么不能在子线程更新 UI？ | ViewRootImpl.checkThread() 检查线程，且 UI 操作非线程安全 |
| PhoneWindow 和 Window 的关系？ | PhoneWindow 是 Window 的唯一实现类 |
| DecorView 什么时候可见？ | ViewRootImpl.setView → WMS.addWindow → performDraw 之后 |

**requestLayout vs invalidate：**

- `requestLayout()`：触发 measure + layout + draw，向上传递 PFLAG_FORCE_LAYOUT
- `invalidate()`：仅触发 draw，标记脏区域
- `postInvalidateOnAnimation()`：下一帧 VSYNC 时 invalidate，避免同一帧多次重绘

**性能优化切入点：**

```
1. 减少 measure 次数：避免嵌套 RelativeLayout（触发两次 measure），用 ConstraintLayout
2. 避免过度绘制：移除不必要的背景色，使用 canvas.clipRect() 裁剪
3. 利用硬件加速：频繁变化的 View 设置 LAYER_TYPE_HARDWARE 缓存为纹理
4. 减少布局层级：ViewStub 延迟加载、merge 标签减少层级
5. 异步 inflate：AsyncLayoutInflater 避免主线程阻塞
```

**硬件加速不支持的操作：**

```kotlin
// 以下操作在硬件加速下可能异常，需回退软件渲染
setLayerType(View.LAYER_TYPE_SOFTWARE, null)
// - Canvas.drawBitmapMesh()
// - Paint.setMaskFilter()（部分）
// - Canvas.drawVertices()
```

### 延伸问题

- [[View绘制三大流程]] — measure/layout/draw 的详细源码
- [[WMS窗口管理与Surface体系]] — WMS 窗口管理与 Surface 体系
- [[事件分发机制]] — View 树遍历顺序与绘制顺序的关系
- [[Activity生命周期回调详解]] — onResume 后 ViewRootImpl 创建时机
- [[Activity所在进程的启动过程]] — 进程启动到 Activity 显示的完整链路
- [[SurfaceView vs TextureView]] — 特殊的 Surface 使用方式
- [[Compose重组原理与性能优化]] — Compose 如何跳过传统 View 绘制流程

## 记忆锚点

Activity 显示六步走：PhoneWindow（壳）→ DecorView（树）→ ViewRootImpl（桥）→ performTraversals（测布局绘）→ Surface/BufferQueue（画板）→ SurfaceFlinger（投影仪）。硬件加速把绘制命令录成 DisplayList，RenderThread 异步交给 GPU，主线程不等渲染完成。
