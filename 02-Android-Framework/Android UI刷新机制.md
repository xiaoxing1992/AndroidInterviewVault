---
tags: [android, 面试, framework, ui, 渲染, vsync, choreographer, S]
difficulty: S
frequency: high
---

# Android UI 刷新机制

## 问题

> Android 的 UI 刷新机制是怎样的？从 invalidate 到屏幕刷新经历了哪些步骤？VSYNC 的作用是什么？同步屏障和异步消息是怎么回事？

## 核心答案

UI 刷新链路：invalidate/requestLayout → ViewRootImpl.scheduleTraversals() → 插入同步屏障 + Choreographer 注册 VSYNC 回调 → VSYNC 到来 → Choreographer.doFrame()（Input→Animation→Traversal→Commit）→ performTraversals（measure→layout→draw）→ RenderThread 异步 GPU 渲染 → 提交 Buffer 到 BufferQueue → SurfaceFlinger 合成 → Display 刷新。从 invalidate 到屏幕显示需要 2-3 个 VSYNC 周期。同步屏障保证 VSYNC 回调（异步消息）优先于普通消息执行，机制是 MessageQueue.next() 遇到屏障时跳过所有同步消息，只处理异步消息。

## 深入解析

### 原理层（源码级）

**整体架构：**

```
                    VSYNC 信号 (硬件中断，来自 Display)
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    VSYNC-app       VSYNC-sf      Display 刷新
   (通知 App 开始    (通知 SF 开始   (显示上一帧)
    绘制下一帧)       合成)

   App 进程           SurfaceFlinger         Display
  ─────────          ───────────────         ───────
  主线程绘制          合成所有 Layer          刷新屏幕
  RenderThread        HWC 硬件合成           像素更新
```

---

#### 第 0 步：触发刷新

```kotlin
// 三种触发方式：

// ① View 内容变化 → invalidate()
textView.text = "Hello"  // TextView.setText 内部调用 invalidate()

// ② View 大小/位置变化 → requestLayout()
view.layoutParams = newLayoutParams  // 内部调用 requestLayout()

// ③ 显示新 View → ViewRootImpl.requestLayout()
wm.addView(decorView)  // 首次显示
```

```java
// View.java
public void invalidate() {
    invalidate(true);
}

void invalidate(boolean invalidateCache) {
    final AttachInfo ai = mAttachInfo;
    if (ai != null) {
        // 向上传递脏区域到 ViewRootImpl
        final Rect damage = ai.mTmpInvalRect;
        damage.set(0, 0, mRight - mLeft, mBottom - mTop);
        p.invalidateChild(this, damage);
        // → ViewRootImpl.invalidateChildInParent()
        //   → scheduleTraversals()
    }
}

public void requestLayout() {
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;
    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
        // 最终到达 ViewRootImpl.requestLayout()
    }
}
```

```
invalidate() vs requestLayout()：

invalidate():
  → 标记 View 为 PFLAG_DIRTY
  → 向上传递到 ViewRootImpl
  → 只触发 draw（不重新 measure/layout）
  → 适合：内容变化但大小不变

requestLayout():
  → 标记 View 为 PFLAG_FORCE_LAYOUT
  → 向上传递到 ViewRootImpl
  → 触发 measure + layout + draw
  → 适合：大小或位置变化
```

---

#### 第 1 步：scheduleTraversals — 等待 VSYNC

```java
// ViewRootImpl.java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;

        // ① 插入同步屏障（阻塞普通消息，优先处理异步消息）
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();

        // ② 向 Choreographer 注册回调（不是立即执行！）
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL,
            mTraversalRunnable,
            null);

        // ③ Choreographer 内部向底层注册 VSYNC 监听
        // → DisplayEventReceiver.requestNextVsync()
        // → 等待下一个 VSYNC 信号到来
    }
}
```

**为什么不立即执行 performTraversals？**

```
原因 1：合并多次刷新请求
  → 一帧内可能有多次 invalidate/requestLayout
  → 如果每次都立即执行，会重复绘制
  → 合并到下一个 VSYNC 统一处理，只绘制一次

原因 2：与屏幕刷新节奏同步
  → 屏幕按 VSYNC 周期刷新（16.6ms）
  → 如果不在 VSYNC 时刻提交 Buffer → 画面撕裂（tearing）

原因 3：同步屏障保证绘制优先
  → postSyncBarrier() 阻塞普通 Message
  → VSYNC 回调是异步消息，不受阻塞
  → 保证绘制任务优先于其他消息执行
```

---

#### 第 2 步：Choreographer 调度回调

```java
// Choreographer.java
// VSYNC 信号到来时，底层回调：
void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
    // 通过 Handler 投递到主线程
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);  // ★ 异步消息，可以穿越同步屏障
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}

// 主线程收到后执行：
void doFrame(long frameTimeNanos, int frame) {
    // ① 检查掉帧
    final long jitterNanos = startNanos - frameTimeNanos;
    if (jitterNanos >= mFrameIntervalNanos) {
        final long skippedFrames = jitterNanos / mFrameIntervalNanos;
        if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
            Log.i(TAG, "Skipped " + skippedFrames + " frames!");
        }
    }

    // ② 按顺序执行回调
    doCallbacks(CALLBACK_INPUT, frameTimeNanos);          // 输入事件
    doCallbacks(CALLBACK_ANIMATION, frameTimeNanos);       // 动画
    doCallbacks(CALLBACK_INSETS_ANIMATION, frameTimeNanos);// 窗口动画
    doCallbacks(CALLBACK_TRAVERSAL, frameTimeNanos);       // ★ View 三大流程
    doCallbacks(CALLBACK_COMMIT, frameTimeNanos);          // 提交
}
```

```
Choreographer 回调顺序及意义：
  CALLBACK_INPUT       → 处理触摸事件（先更新交互状态）
  CALLBACK_ANIMATION   → 属性动画计算（更新动画属性）
  CALLBACK_TRAVERSAL   → performTraversals（measure→layout→draw）
  CALLBACK_COMMIT      → 提交帧数据

为什么 Input 在 Traversal 之前？
  → 先处理触摸事件，更新 View 状态
  → 再执行绘制，确保绘制结果包含最新的触摸响应
```

---

#### 第 3 步：performTraversals — 主线程三大流程

```java
// ViewRootImpl.java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;

        // 移除同步屏障（让普通消息恢复执行）
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        // 执行绘制
        performTraversals();
    }
}

private void performTraversals() {
    // ① 预处理：检查窗口大小/可见性变化
    // → 如有变化 → relayoutWindow() → 向 WMS 请求新的 Surface 参数

    // ② 测量
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    // → DecorView.measure() → 递归测量所有子 View

    // ③ 布局
    performLayout(lp, mWidth, mHeight);
    // → DecorView.layout() → 递归布局所有子 View

    // ④ 绘制
    performDraw();
}
```

**硬件加速下 performDraw 的流程：**

```
performDraw()
  → draw()
    → ThreadedRenderer.draw(mView, ...)
      → updateRootDisplayList(mView, ...)
        → mView.updateDisplayListIfDirty()
          → 遍历 View 树，更新脏 View 的 DisplayList
          → 不脏的 View 复用上次的 DisplayList（局部更新）
      → syncAndDrawFrame()
        → 把 DisplayList 同步给 RenderThread
        → 返回（主线程释放，可以处理下一个 Message）
```

---

#### 第 4 步：RenderThread 异步渲染

```
主线程                           RenderThread
──────                          ─────────────
构建 DisplayList
  → syncAndDrawFrame()
  → 释放主线程           ──→    遍历 DisplayList
                                → 转换为 GPU 指令（OpenGL/Vulkan）
                                → 执行 GPU 渲染
                                → eglSwapBuffers()
                                  → 把渲染结果提交到 BufferQueue

RenderThread 的意义：
  → 主线程只负责构建 DisplayList（CPU 计算）
  → RenderThread 负责 GPU 渲染（GPU 计算）
  → 两者并行：主线程处理下一帧的 Input/Animation，同时 RenderThread 渲染当前帧
```

---

#### 第 5-6 步：BufferQueue → SurfaceFlinger → Display

```
App (Producer)              BufferQueue              SurfaceFlinger (Consumer)
─────────────              ───────────              ────────────────────────
dequeueBuffer() ←───────── 空闲 Buffer
绘制内容到 Buffer
queueBuffer() ──────────→ 已填充 Buffer ──────────→ acquireBuffer()
                                                    合成所有 Layer
                                                    HWC → Display → 刷新

Triple Buffering（3 个 Buffer 轮转）：
  Buffer A: 正在显示
  Buffer B: SurfaceFlinger 正在合成
  Buffer C: App 正在绘制
  → 三者并行，互不阻塞
```

---

### 同步屏障与异步消息详解

#### 什么是同步屏障？

```
MessageQueue 中有三种消息：
  ① 普通同步消息：Handler.sendMessage() 发的（默认）
  ② 异步消息：msg.setAsynchronous(true) 标记的
  ③ 同步屏障：postSyncBarrier() 插入的特殊 Message（target == null）

同步屏障的作用：
  → 插入 MessageQueue 后，MessageQueue.next() 的行为改变
  → 遇到屏障 → 跳过所有同步消息，只处理异步消息
  → 直到 removeSyncBarrier() 移除屏障后，同步消息才恢复执行
```

#### 同步屏障源码解析

```java
// MessageQueue.java

// ★ 插入同步屏障
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // 创建一个特殊的 Message：target == null
    Message msg = Message.obtain();
    msg.markInUse();
    msg.when = when;
    msg.arg1 = token;  // token 用于后续 removeSyncBarrier

    synchronized (this) {
        // 按时间插入链表
        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) {
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
    // 注意：这个 Message 的 target == null
    // 这就是它和普通消息的本质区别
}
```

#### MessageQueue.next() 如何处理屏障

```java
// MessageQueue.java — next() 核心逻辑（简化）
Message next() {
    int nextPollTimeoutMillis = 0;

    for (;;) {
        nativePollOnce(mPtr, nextPollTimeoutMillis);  // epoll_wait

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;

            // ★ 关键：检查是否是同步屏障
            if (msg != null && msg.target == null) {
                // msg.target == null → 这是一个同步屏障！

                // ★ 跳过所有同步消息，找到第一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
                // ↑ 循环条件：msg 存在 且 不是异步消息 → 继续跳过
                // 直到找到一个异步消息 或 遍历完链表

                // 如果 msg == null → 没有异步消息可处理 → 继续等待
            }

            if (msg != null) {
                if (now < msg.when) {
                    // 消息还没到时间，设置超时
                    nextPollTimeoutMillis = (int) Math.min(
                        msg.when - now, Integer.MAX_VALUE);
                } else {
                    // ★ 取出消息返回（同步消息直接返回，异步消息穿越屏障后返回）
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

#### 完整的消息处理流程图

```
场景 1：没有同步屏障（正常情况）
  MessageQueue: [sync1] → [sync2] → [async3] → [sync4]
  next() 处理顺序：sync1 → sync2 → async3 → sync4
  → 所有消息按时间顺序处理

场景 2：有同步屏障
  MessageQueue: [barrier] → [sync1] → [sync2] → [async3] → [sync4]
  next() 处理顺序：
    → 遇到 barrier（target == null）
    → 跳过 sync1（同步，跳过）
    → 跳过 sync2（同步，跳过）
    → 找到 async3（异步！）→ 取出返回
    → 下次 next()：跳过 sync4（同步，还是被屏障挡住）
    → 直到 removeSyncBarrier() 移除屏障
    → sync1、sync2、sync4 恢复按顺序处理
```

#### ViewRootImpl 中的完整使用

```java
// ViewRootImpl.java

// 需要绘制时
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;

        // ① 插入同步屏障 → 阻塞所有普通消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();

        // ② 注册 VSYNC 回调（异步消息）
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL,
            mTraversalRunnable,
            null);
        // Choreographer 内部：
        //   → 向底层注册 VSYNC 监听
        //   → VSYNC 到来时发送异步消息（msg.setAsynchronous(true)）
        //   → 这个异步消息可以穿越同步屏障
    }
}

// VSYNC 到来 → 异步消息穿越屏障 → doFrame() → performTraversals()
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;

        // ③ 移除同步屏障 → 普通消息恢复执行
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        // ④ 执行绘制
        performTraversals();
    }
}
```

```
时序：
  scheduleTraversals()
    → postSyncBarrier()              ← 插入屏障，普通消息被阻塞
    → Choreographer.postCallback()   ← 注册 VSYNC（异步）

  此时 MessageQueue 中：
    [barrier] → [普通消息A] → [普通消息B] → [普通消息C]
    → 普通消息 A/B/C 全部被屏障挡住，不会执行

  VSYNC 到来
    → Choreographer 发送异步消息（msg.setAsynchronous(true)）
    → MessageQueue 变为：
        [barrier] → [普通消息A] → [普通消息B] → [async_vsync] → [普通消息C]
    → next() 跳过普通消息 A/B，找到 async_vsync → 返回
    → doFrame() → performTraversals() → 绘制完成

  doTraversal()
    → removeSyncBarrier()            ← 移除屏障
    → 普通消息 A/B/C 恢复按顺序执行
```

#### 为什么需要同步屏障？

```
没有同步屏障时：
  MessageQueue: [普通消息A] → [普通消息B] → [VSYNC异步消息]
  → 如果普通消息 A 处理耗时 10ms，B 耗时 8ms
  → VSYNC 消息要等 18ms 后才执行
  → 已经超过一个 VSYNC 周期 → 丢帧

有同步屏障时：
  MessageQueue: [barrier] → [普通消息A] → [普通消息B] → [VSYNC异步消息]
  → next() 直接跳过 A 和 B，优先处理 VSYNC 异步消息
  → VSYNC 消息立即执行 → 不丢帧
  → 等绘制完成（removeSyncBarrier）后，A 和 B 再按顺序执行

核心目的：保证 UI 绘制（VSYNC 回调）优先于其他消息执行
```

#### 异步消息能穿越屏障的根本原因

```java
// MessageQueue.next() 中的核心判断：

if (msg != null && msg.target == null) {
    // ★ 遇到同步屏障（target == null）
    // 找到第一个异步消息
    do {
        prevMsg = msg;
        msg = msg.next;
    } while (msg != null && !msg.isAsynchronous());
    // ↑ 关键：只要 msg 不是异步消息，就继续跳过
    // ↓ 只有异步消息（isAsynchronous() == true）才会被选中
}

// Message.java
public boolean isAsynchronous() {
    return (flags & FLAG_ASYNCHRONOUS) != 0;
    // setAsynchronous(true) 设置了这个 flag
}
```

```
本质就是 MessageQueue.next() 中的一个 if 判断：

  遇到 barrier（target==null）：
    → while 循环跳过所有同步消息
    → 只有 isAsynchronous() == true 的消息才会被返回

  没有 barrier：
    → 按时间顺序处理所有消息，不分同步异步

所以"穿越同步屏障"不是什么魔法，就是 MessageQueue.next() 的一个分支逻辑。
```

---

### 完整时序图

```
时间轴 →

VSYNC-app    VSYNC-sf    VSYNC-display
    │            │             │
    ▼            ▼             ▼

App 主线程:   [Input][Anim][measure][layout][draw]
                                          │
App RenderThread:                          [GPU渲染][提交Buffer]
                                                    │
SurfaceFlinger:                                      [合成]
                                                          │
Display:                                                   [刷新]

Frame N 的生命周期：
  T0: VSYNC-app → 主线程开始
  T1: 主线程完成 → RenderThread 开始
  T2: RenderThread 提交 Buffer
  T3: VSYNC-sf → SurfaceFlinger 合成
  T4: VSYNC-display → 屏幕显示

从 invalidate 到屏幕显示：2-3 个 VSYNC 周期（~33-50ms @60Hz）
```

### 实战层（项目经验）

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| invalidate 和 requestLayout 区别？ | invalidate 只触发 draw，requestLayout 触发 measure+layout+draw |
| 为什么不立即执行绘制？ | 合并多次请求、与 VSYNC 同步防止撕裂、同步屏障保证优先 |
| VSYNC 的作用？ | 同步 App 绘制、SurfaceFlinger 合成、Display 刷新的节奏 |
| 同步屏障是什么？ | 插入 MessageQueue 的特殊标记（target==null），阻塞同步消息，只处理异步消息 |
| 异步消息为什么能穿越屏障？ | MessageQueue.next() 中遇到屏障时 while 循环跳过同步消息，只有 isAsynchronous()==true 才被选中 |
| 谁发送异步消息？ | Choreographer 在 VSYNC 到来时发送（msg.setAsynchronous(true)） |
| Choreographer 做什么？ | VSYNC 驱动的帧调度器，按 Input→Animation→Traversal→Commit 顺序执行回调 |
| RenderThread 的意义？ | 主线程构建 DisplayList，RenderThread 异步 GPU 渲染，两者并行 |
| 从 invalidate 到屏幕刷新要多久？ | 2-3 个 VSYNC 周期（~33-50ms @60Hz） |
| Triple Buffering 的作用？ | 3 个 Buffer 轮转，App/SF/Display 并行，减少掉帧 |
| DisplayList 的好处？ | 局部更新：只重建脏 View 的 RenderNode，其他复用 |

### 延伸问题

- [[View绘制三大流程]] — measure/layout/draw 的详细源码
- [[WMS窗口管理与Surface体系]] — Surface 分配和窗口管理
- [[Handler-Looper-MessageQueue机制]] — 同步屏障的基础
- [[卡顿优化-Choreographer与掉帧检测]] — 丢帧检测与优化
- [[Activity显示原理]] — Activity 到屏幕的完整显示链路
- [[丢帧原因与检测方案详解]] — 丢帧原因全景

## 记忆锚点

UI 刷新七步走：触发 → 等 VSYNC（同步屏障护航）→ Choreographer 调度 → 三大流程 → RenderThread 渲染 → BufferQueue → SurfaceFlinger 合成 → 屏幕。同步屏障 = 路障挡住普通消息，只有异步消息（VSYNC 回调）能穿越，保证绘制不被其他消息耽误。穿越不是魔法，就是 next() 里的一个 while 循环跳过同步消息。
