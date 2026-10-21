---
tags: [android, 面试, framework, vsync, choreographer, display, S]
difficulty: S
frequency: high
---

# VSYNC 详解

## 问题

> VSYNC 是什么？Android 中有几路 VSYNC？为什么需要 VSYNC？VSYNC 是每个都发给 App 吗？Choreographer 和 VSYNC 是什么关系？

## 核心答案

VSYNC（垂直同步）是 Display 控制器发出的周期性信号，频率 = 屏幕刷新率（60Hz = 16.6ms）。Android 有三路 VSYNC：VSYNC-app（通知 App 绘制）、VSYNC-sf（通知 SurfaceFlinger 合成）、VSYNC-display（通知 Display 刷新），三者错开半个周期实现流水线并行。VSYNC 是按需请求的，App 没有 UI 变化时不请求也不收到 VSYNC，节省电量。Choreographer 是 VSYNC 驱动的帧调度器，收到 VSYNC 后按 Input→Animation→Traversal→Commit 顺序执行回调。VSYNC 回调通过异步消息 + 同步屏障保证优先执行。

## 深入解析

### 原理层（源码级）

**为什么需要 VSYNC？**

```
没有 VSYNC 时：
  App 随时提交 Buffer → SurfaceFlinger 随时合成 → Display 随时刷新
  → 问题：Display 正在扫描上半屏时，Buffer 被替换
  → 上半屏显示旧帧，下半屏显示新帧
  → 画面撕裂（Tearing）

有 VSYNC 时：
  所有操作都对齐到 VSYNC 信号
  → App 在 VSYNC-app 时刻开始绘制
  → SurfaceFlinger 在 VSYNC-sf 时刻开始合成
  → Display 在 VSYNC-display 时刻切换 Buffer
  → 三者节奏一致，不会撕裂
```

---

### 三路 VSYNC

```
硬件 VSYNC 信号（来自 Display 控制器）
  │
  ├──→ VSYNC-app    通知 App 开始绘制下一帧
  ├──→ VSYNC-sf     通知 SurfaceFlinger 开始合成
  └──→ VSYNC-display 通知 Display 切换 Buffer 显示

为什么要分三路？
  → 错开 App 和 SF 的执行时间
  → App 绘制和 SF 合成可以流水线并行
  → 避免两者争抢 CPU/GPU 资源
```

```
60Hz 下的 VSYNC 时序（16.6ms 一帧）：

时间 →  0ms      16.6ms    33.3ms    50ms
VSYNC-app:  |          |          |          |
             App 开始    App 开始    App 开始
             绘制 Frame1  绘制 Frame2  绘制 Frame3

VSYNC-sf:        8.3ms      25ms       41.6ms
                  |          |          |
                SF 开始      SF 开始     SF 开始
                合成 Frame1  合成 Frame2  合成 Frame3

VSYNC-display:  16.6ms     33.3ms     50ms
                  |          |          |
                Display     Display    Display
                显示 Frame1  显示 Frame2  显示 Frame3

VSYNC-app 和 VSYNC-sf 错开半个周期
→ App 绘制完一帧后，SF 立刻合成
→ 流水线并行，效率最高
```

**三帧流水线：**

```
时间轴 →

VSYNC-app:    |  Frame1   |  Frame2   |  Frame3   |
                [绘制 F1]   [绘制 F2]   [绘制 F3]

VSYNC-sf:          |  Frame1   |  Frame2   |  Frame3   |
                     [合成 F1]   [合成 F2]   [合成 F3]

Display:                 |  Frame1   |  Frame2   |
                          [显示 F1]   [显示 F2]

App 绘制和 SF 合成并行进行
→ Frame2 的绘制和 Frame1 的合成同时进行
→ 效率翻倍
```

---

### VSYNC 传递链路

```
硬件层：Display 控制器
  │
  ↓ 硬件中断
HAL层：HWComposer
  │
  ↓
Native层：DispSync / VsyncReactor
  │
  ├──→ VSyncDispatch
  │      ├──→ AppEventThread → VSYNC-app
  │      └──→ SFEventThread  → VSYNC-sf
  │
  ↓
Framework层：
  ├── VSYNC-app → Choreographer（App 进程）
  └── VSYNC-sf  → SurfaceFlinger
```

---

#### VSYNC-app 的传递（到 App 进程）

```
SurfaceFlinger 的 EventThread
  → 通过 BitTube（socket pair）发送 VSYNC 事件
  → App 进程的 DisplayEventReceiver 收到
    → native 层：NativeDisplayEventReceiver.handleEvent()
      → JNI 回调 Java 层
        → DisplayEventReceiver.onVsync()
          → Choreographer.onVsync()
            → 发送异步 Message 到主线程
              → Choreographer.doFrame()
```

```java
// Choreographer.java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {

    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
        // ★ 通过 Handler 发送异步消息到主线程
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);  // 异步消息，穿越同步屏障
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        // 主线程收到后执行 doFrame
        doFrame(mTimestampNanos, mFrame);
    }
}
```

---

### VSYNC 按需请求机制

**不是每个 VSYNC 都发给 App！**

```java
// Choreographer.java
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            scheduleVsyncLocked();
        }
    }
}

private void scheduleVsyncLocked() {
    // 调用 native 方法请求下一个 VSYNC
    mDisplayEventReceiver.scheduleVsync();
    // → nativeScheduleVsync()
    //   → NativeDisplayEventReceiver.requestNextVsync()
    //     → EventThread.requestNextVsync()
    //       → 把当前 App 加入 VSYNC 等待列表
    //       → 下一个 VSYNC 到来时通知这个 App
}
```

```
请求 VSYNC 的条件：
  → invalidate() 被调用（View 内容变化）
  → requestLayout() 被调用（View 大小/位置变化）
  → 动画在播放（Choreographer.postFrameCallback）
  → ViewRootImpl.scheduleTraversals() 被调用

执行后：
  doFrame() 执行完毕
  → mFrameScheduled = false
  → 从 VSYNC 等待列表移除
  → 后续 VSYNC 不再发给这个 App（除非再次请求）

为什么按需？
  → App 没有 UI 变化时不需要 VSYNC
  → 避免无意义的唤醒，节省电量
  → 后台 App 不消耗 VSYNC 资源
```

```
场景：
  App 静止不动（没有动画、没有触摸）
    → 不请求 VSYNC → 不收到 VSYNC → 不消耗 CPU/GPU → 省电

  用户触摸屏幕
    → Input 事件触发 → requestNextVsync()
    → 收到 VSYNC → doFrame() → 处理 Input + 绘制
    → 如果没有后续触摸 → 不再请求 → 停止接收 VSYNC

  动画播放中
    → 每帧 doFrame() 结束后再次 requestNextVsync()
    → 持续收到 VSYNC → 持续绘制 → 动画流畅
    → 动画结束 → 不再请求 → 停止接收 VSYNC
```

---

### Choreographer.doFrame — VSYNC 到来后做什么

```java
// Choreographer.java
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos = System.nanoTime();

    // ① 检查掉帧
    final long jitterNanos = startNanos - frameTimeNanos;
    if (jitterNanos >= mFrameIntervalNanos) {
        final long skippedFrames = jitterNanos / mFrameIntervalNanos;
        if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
            Log.i(TAG, "Skipped " + skippedFrames + " frames!");
        }
        // 调整 frameTimeNanos 为最近的 VSYNC 时间点
        long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
        frameTimeNanos = startNanos - lastFrameOffset;
    }

    // ② 按顺序执行回调
    doCallbacks(CALLBACK_INPUT, frameTimeNanos);          // 输入事件
    doCallbacks(CALLBACK_ANIMATION, frameTimeNanos);       // 动画
    doCallbacks(CALLBACK_INSETS_ANIMATION, frameTimeNanos);// 窗口动画
    doCallbacks(CALLBACK_TRAVERSAL, frameTimeNanos);       // View 三大流程
    doCallbacks(CALLBACK_COMMIT, frameTimeNanos);          // 提交帧数据
}
```

```
doFrame 回调顺序及意义：

  CALLBACK_INPUT (0)
    → 处理触摸事件
    → InputEventReceiver 收到的事件在这里派发

  CALLBACK_ANIMATION (1)
    → 属性动画计算（ValueAnimator、ObjectAnimator）
    → Choreographer.postFrameCallback() 注册的回调在这里执行

  CALLBACK_INSETS_ANIMATION (2)
    → 窗口 insets 动画（状态栏、导航栏变化）

  CALLBACK_TRAVERSAL (3)
    → performTraversals()（measure → layout → draw）
    → ViewRootImpl.scheduleTraversals() 注册的回调

  CALLBACK_COMMIT (4)
    → 提交帧数据
    → 调整帧时间戳
    → 如果动画有延迟，记录偏移量

为什么 Input 在 Traversal 之前？
  → 先处理触摸事件，更新 View 状态
  → 再执行绘制，确保绘制结果包含最新的触摸响应
```

---

### VSYNC 与同步屏障的配合

```
scheduleTraversals() 中：
  ① postSyncBarrier()     ← 插入同步屏障，阻塞普通消息
  ② postCallback(TRAVERSAL) ← 注册 VSYNC 回调

VSYNC 到来时：
  Choreographer.onVsync()
    → msg.setAsynchronous(true)   ← 异步消息
    → sendMessageAtTime()

MessageQueue.next() 中：
  → 遇到同步屏障（target == null）
  → 跳过所有同步消息
  → 找到这个异步消息（VSYNC 回调）→ 返回
  → doFrame() 执行
  → performTraversals() 执行

doTraversal() 中：
  → removeSyncBarrier()   ← 移除屏障
  → 普通消息恢复执行

意义：保证 VSYNC 回调（UI 绘制）不被其他普通消息延迟
```

---

### 同步屏障 vs VSYNC 的区别

```
同步屏障（SyncBarrier）：
  → 是 MessageQueue 的一个机制
  → 一个特殊的 Message（target == null）
  → 作用：阻塞普通同步消息，让异步消息优先执行
  → 任何线程、任何场景都能用

VSYNC：
  → 是 Display 硬件发出的周期性信号
  → 由 SurfaceFlinger 的 EventThread 分发给 App
  → 作用：通知 App "可以开始绘制下一帧了"
  → 是硬件层面的概念

两者不是同一个东西，但在 UI 渲染流程中配合使用：
  VSYNC = "信号"（触发绘制的时机）
  同步屏障 = "路障"（保证绘制消息不被其他消息挡住）
```

```
它们的关系：

  VSYNC 到来
    → Choreographer 发送异步消息（msg.setAsynchronous(true)）
    → 需要穿越同步屏障
    → 保证绘制优先执行

类比：
  VSYNC = 上课铃声（告诉你该上课了）
  同步屏障 = 教室门口的保安（只让学生进，拦住其他人）
  异步消息 = 学生的通行证（可以穿越保安）

  铃声响了（VSYNC）→ 学生拿着通行证（异步消息）→ 保安放行（穿越同步屏障）→ 进教室上课（执行绘制）
```

```
没有同步屏障会怎样？

  MessageQueue: [消息A(5ms)] → [消息B(10ms)] → [VSYNC消息]
  → VSYNC 消息要等 A(5ms) + B(10ms) = 15ms 后才执行
  → 超过 16.6ms 的时间预算 → 可能丢帧

  有同步屏障：
  MessageQueue: [barrier] → [消息A] → [消息B] → [VSYNC消息]
  → next() 跳过 A 和 B，直接处理 VSYNC 消息
  → 绘制立即执行 → 不丢帧
```

| 维度 | 同步屏障 | VSYNC |
|------|---------|-------|
| 是什么 | MessageQueue 机制 | Display 硬件信号 |
| 作用 | 阻塞普通消息，让异步消息优先 | 触发绘制时机 |
| 位置 | App 进程 MessageQueue | Display → SF → App |
| 关系 | 为 VSYNC 回调"开路" | 被同步屏障"护送" |
| 谁创建 | ViewRootImpl.postSyncBarrier() | Display 硬件 → EventThread |
| 谁移除 | ViewRootImpl.removeSyncBarrier() | 不需要移除（周期性信号） |

---

### Native 层实现

```cpp
// DispSync.cpp — VSYNC 信号的软件预测和分发
class DispSync {
    // 收到硬件 VSYNC 后，用模型预测后续的 VSYNC 时间点
    // 即使硬件 VSYNC 有抖动，软件预测也能保持稳定

    // 分发给两个 EventThread：
    //   mEventThread[0] → AppEventThread（VSYNC-app）
    //   mEventThread[1] → SFEventThread（VSYNC-sf）
};

// EventThread.cpp
class EventThread : public Thread {
    void requestNextVsync(const sp<BitTube>& connection) {
        // 把这个连接加入 VSYNC 等待列表
        // 下一个 VSYNC 到来时通知
    }

    void onVSyncEventReceived(nsecs_t timestamp) {
        // 遍历等待列表
        // 通过 BitTube 发送 VSYNC 事件给每个订阅者
        for (auto& connection : mDisplayEventConnections) {
            Event vsync;
            vsync.header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
            vsync.header.timestamp = timestamp;
            connection->postEvent(vsync);
        }
    }
};

// BitTube.cpp — App 和 EventThread 之间的通信通道
class BitTube {
    // 本质是一对 socket pair（Unix domain socket）
    // EventThread 写入 → App 读取
    // 高效、低延迟
};
```

---

### Choreographer 的其他回调

```java
// 除了 VSYNC 驱动的回调，Choreographer 还支持手动投递回调

// ① postFrameCallback — 每帧执行（常用于自定义动画和 FPS 监控）
Choreographer.getInstance().postFrameCallback { frameTimeNanos ->
    // 每帧 VSYNC 到来时执行
}

// ② postCallback — 只执行一次
Choreographer.getInstance().postCallback(
    Choreographer.CALLBACK_ANIMATION,
    { /* 动画更新 */ },
    null
)

// ③ postCallbackDelayed — 延迟执行
Choreographer.getInstance().postCallbackDelayed(
    Choreographer.CALLBACK_ANIMATION,
    { /* 延迟执行 */ },
    null,
    100 // 延迟 100ms
)
```

---

### 实战层（项目经验）

**不同刷新率的影响：**

```
60Hz  → 16.6ms/帧 → 主线程每帧最多 16.6ms
90Hz  → 11.1ms/帧 → 主线程每帧最多 11.1ms
120Hz → 8.3ms/帧  → 主线程每帧最多 8.3ms

刷新率越高，每帧时间预算越少，越容易掉帧
→ 120Hz 屏幕对性能要求更高
→ 可以通过 WindowManager.LayoutParams.preferredDisplayModeId 设置刷新率
```

**VSYNC 与电量：**

```
VSYNC 的按需请求机制是电量优化的关键：
  → 没有 UI 变化 → 不请求 VSYNC → 不唤醒 App → 省电
  → 动画播放中 → 持续请求 VSYNC → 持续消耗 CPU/GPU → 费电
  → 后台 App → 不应该请求 VSYNC

优化建议：
  → 动画结束时及时停止 Choreographer 回调
  → 后台时不要注册 FrameCallback
  → 使用 ValueAnimator 而不是手动 postFrameCallback（会自动管理生命周期）
```

**面试高频追问：**

| 问题                            | 答案                                                                |
| ----------------------------- | ----------------------------------------------------------------- |
| VSYNC 是什么？                    | Display 控制器发出的周期性信号，频率 = 屏幕刷新率                                    |
| 为什么需要 VSYNC？                  | 同步 App/SF/Display 的节奏，防止画面撕裂                                      |
| Android 有几路 VSYNC？            | 三路：VSYNC-app、VSYNC-sf、VSYNC-display，错开半个周期                        |
| VSYNC-app 和 VSYNC-sf 为什么要错开？  | 流水线并行，App 绘制和 SF 合成同时进行                                           |
| VSYNC 是每个都发给 App 吗？           | 不是，按需请求（scheduleVsync），无 UI 变化时不发                                 |
| App 怎么请求 VSYNC？               | Choreographer.scheduleVsync() → nativeScheduleVsync → EventThread |
| Choreographer 做什么？            | VSYNC 驱动的帧调度器，按 Input→Animation→Traversal→Commit 执行回调             |
| VSYNC 到来后怎么保证立即执行？            | 异步消息 + 同步屏障，穿越普通消息优先执行                                            |
| 没有动画时 App 还收到 VSYNC 吗？        | 不收到，按需请求机制，没有 UI 变化就取消订阅                                          |
| Triple Buffering 和 VSYNC 的关系？ | 3 个 Buffer 轮转对齐 VSYNC 节奏，保证 App/SF/Display 并行                     |
| 120Hz 对性能有什么影响？               | 每帧只有 8.3ms，主线程耗时操作更容易掉帧                                           |

### 延伸问题

- [[Android UI刷新机制]] — VSYNC 驱动的完整 UI 刷新链路
- [[Activity显示原理]] — Activity 到屏幕的完整显示链路
- [[丢帧原因与检测方案详解]] — VSYNC 周期内超时导致的丢帧
- [[卡顿优化-Choreographer与掉帧检测]] — Choreographer 掉帧检测
- [[Handler-Looper-MessageQueue机制]] — 同步屏障与异步消息的基础
- [[Surface绘制原理]] — VSYNC 驱动的 Buffer 提交与合成

## 记忆锚点

VSYNC = 屏幕的"节拍器"，每 16.6ms（60Hz）敲一下。三路分工：App 画（VSYNC-app）、SF 合（VSYNC-sf）、Display 显示（VSYNC-display），错开半个周期流水线并行。按需敲：App 有活干才敲，没活不敲，省电。Choreographer 是"指挥家"，节拍一响就指挥 Input→Animation→Traversal→Commit 按顺序上场。
