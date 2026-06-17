---
tags: [android, 面试, framework, wms, surface, window, S]
difficulty: S
frequency: high
---

# WMS 窗口管理与 Surface 体系

## 问题

> WMS（WindowManagerService）的职责是什么？在 Activity 显示过程中发挥了什么作用？Window、Surface、SurfaceFlinger 之间的关系是怎样的？

## 核心答案

WMS 运行在 system_server 进程，是窗口层的核心管理者，负责五大职责：① 窗口注册与管理（addWindow/removeWindow）② Surface 分配（relayoutWindow）③ Z-order 层级管理 ④ 输入事件路由（根据坐标和 Z-order 决定事件发给哪个窗口）⑤ 窗口布局与动画。WMS 不参与 View 的 measure/layout/draw（这些在 App 进程内完成），但在窗口注册、Surface 分配、输入事件分发三个关键节点介入。App 绘制完成后通过 BufferQueue（mmap 共享内存）直接与 SurfaceFlinger 通信，不经过 WMS。

## 深入解析

### 原理层（源码级）

**显示体系层次：**

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

### WMS 的五大职责

#### 1. 窗口注册与管理

```java
// App 调用 wm.addView(decor) 的完整链路：
WindowManagerGlobal.addView()
  → ViewRootImpl.setView()
    → mWindowSession.addToDisplayAsUser(mWindow, attrs, ...)
    // mWindowSession 是 IWindowSession 的 Binder Proxy
    // mWindow 是 W extends IWindow.Stub（ViewRootImpl 暴露给 WMS 的接口）

// WMS 侧收到后：
// Session.java (system_server)
public int addToDisplayAsUser(IWindow window, ...) {
    return mService.addWindow(this, window, ...);
}

// WindowManagerService.java
public int addWindow(Session session, IWindow client, ...) {
    // 1. 权限检查
    // 2. Token 合法性校验
    // 3. 创建 WindowState（窗口在 WMS 中的数据结构）
    WindowState win = new WindowState(this, session, client, token, ...);
    // 4. 计算窗口层级 Z-order
    // 5. 建立 InputChannel（输入事件通道）
    // 6. 返回结果
    return WindowManagerGlobal.ADD_OKAY;
}
```

常见错误码：

| 错误码 | 原因 |
|--------|------|
| `ADD_BAD_APP_TOKEN` | Activity 的 token 无效（Activity 已销毁） |
| `ADD_NOT_APP_TOKEN` | token 不属于 Application（类型错误） |
| `ADD_DUPLICATE_ADD` | 同一个 Window 重复添加 |
| `ADD_PERMISSION_DENIED` | 没有权限添加该类型的窗口（如悬浮窗） |

```kotlin
// AlertDialog 用 baseContext 崩溃的根源就在这里
// baseContext 没有 Activity 的 Window Token
// → WMS 检查 token → null → ADD_BAD_APP_TOKEN
// → BadTokenException: "Unable to add window -- token is not for an application"
```

WMS 维护所有窗口的 WindowState 列表：

```
WMS 窗口列表：
  ├── 状态栏窗口 (TYPE_STATUS_BAR, Z=2000)
  ├── 导航栏窗口 (TYPE_NAVIGATION_BAR)
  ├── App Activity 窗口 (TYPE_APPLICATION, Z=2)
  ├── Dialog 窗口 (TYPE_APPLICATION, Z=2，但在 Activity 前面)
  ├── Toast 窗口 (TYPE_TOAST, Z=2005)
  └── 悬浮窗 (TYPE_APPLICATION_OVERLAY, Z=2038)
```

---

#### 2. Surface 分配与管理

```
App 需要绘制内容，必须有 Surface（绘图画布）

第一次 performTraversals 时：
  → ViewRootImpl.relayoutWindow()
    → WMS.relayoutWindow()
      → WMS 为这个窗口创建/更新 Surface
      → 通过 Binder 返回 Surface 给 App

后续绘制：
  → App 直接操作 Surface（lockCanvas / unlockCanvas）
  → 不需要再经过 WMS
  → 通过 BufferQueue 直接与 SurfaceFlinger 通信

WMS 管理 Surface 的生命周期：
  → 窗口创建时分配
  → 窗口大小变化时更新
  → 窗口销毁时释放
```

---

#### 3. Z-order 层级管理

```
WMS 决定每个窗口在屏幕上的前后顺序：

Z-order 从小到大（越大的越靠前）：
  TYPE_BASE_APPLICATION (1)
  TYPE_APPLICATION (2)
  TYPE_APPLICATION_STARTING (3)
  ...
  TYPE_TOAST (2005)
  TYPE_SYSTEM_ALERT (2003)
  TYPE_STATUS_BAR (2000)

同一类型内按添加顺序排列
Dialog 窗口在 Activity 窗口前面
Toast 在最前面

WMS 还负责：
  → 窗口切换时的 Z-order 调整
  → 多窗口/分屏模式下的 TaskStack 排列
```

---

#### 4. 输入事件分发

```
用户触摸屏幕
  → 硬件中断 → InputManagerService (system_server)
    → 根据触摸坐标 + 窗口 Z-order
    → WMS 确定触摸事件应该发给哪个窗口
    → 通过 InputChannel 把事件发给对应的 ViewRootImpl
      → ViewRootImpl.mInputEventReceiver 收到事件
        → InputStage 责任链处理：
          NativePreIme → ViewPreIme → Ime → EarlyPostIme
          → NativePostIme → ViewPostIme → Synthetic
            → DecorView.dispatchTouchEvent()
              → Activity.dispatchTouchEvent()
                → Window.superDispatchTouchEvent()
                  → ViewGroup.dispatchTouchEvent()
                    → View.onTouchEvent()

WMS 的作用：
  → 根据 Z-order 和坐标决定事件发给谁
  → 前面的窗口消费了事件，后面的窗口收不到
  → 多窗口/分屏模式下正确路由
```

---

#### 5. 窗口布局与动画

```
WMS 计算每个窗口的位置和大小：
  → 考虑状态栏、导航栏、刘海屏
  → 考虑多窗口/分屏模式
  → 考虑屏幕旋转
  → 通知 App 窗口大小变化

WMS 管理窗口动画：
  → Activity 切换动画（淡入淡出、滑动）
  → Dialog 弹出动画
  → 窗口缩放动画
  → 通过 AnimationLayer 驱动

窗口变化通知链路：
  WMS 检测到窗口变化
    → 通过 IWindow (Binder) 回调 ViewRootImpl.W
      → ViewRootImpl 收到通知
        → requestLayout() / invalidate()
          → performTraversals()
            → relayoutWindow() → 向 WMS 请求新的窗口参数
            → measure → layout → draw
```

---

### WMS 参与 vs 不参与的部分

```
WMS 参与的关键节点：
  → 窗口注册（addToDisplayAsUser）
  → Surface 分配（relayoutWindow）
  → 输入事件路由（InputManagerService + InputChannel）
  → 窗口动画
  → 窗口布局变化通知

WMS 不参与的部分：
  → measure / layout / draw（App 进程内完成）
  → Surface 内容绘制（App 直接操作 Surface）
  → Buffer 到 SurfaceFlinger（通过 BufferQueue mmap 通信）
```

**WMS 在完整显示链路中的位置：**

```
Activity.onCreate()
  → setContentView()                    ← 创建 View 树
Activity.onResume()
  → WindowManagerGlobal.addView()
    → ViewRootImpl 创建
      → setView()
        → requestLayout()               ← 预约 VSYNC
        → addToDisplayAsUser()           ← WMS: 注册窗口、建立 InputChannel
        → InputEventReceiver 创建

VSYNC 到来
  → performTraversals()
    → relayoutWindow()                  ← WMS: 获取/更新 Surface
    → performMeasure()                  ← App 进程内完成，WMS 不参与
    → performLayout()                   ← App 进程内完成，WMS 不参与
    → performDraw()                     ← App 进程内完成，WMS 不参与
      → Surface.lockCanvas()
      → View 树绘制
      → Surface.unlockCanvas()          ← 提交 Buffer 到 BufferQueue

SurfaceFlinger                          ← 从 BufferQueue 取 Buffer（mmap）
  → 合成所有 Layer
  → HWC → Display → 用户看到画面
```

---

### Surface 与 BufferQueue

```
App (Producer)              BufferQueue              SurfaceFlinger (Consumer)
─────────────              ───────────              ────────────────────────
dequeueBuffer() ←───────── 空闲 Buffer
绘制内容 (Canvas/OpenGL)
queueBuffer() ──────────→ 已填充 Buffer ──────────→ acquireBuffer()
                                                    合成所有 Layer
                                                    releaseBuffer() → 回收
```

- BufferQueue 通常有 3 个 Buffer（Triple Buffering）
- App 绘制和 SurfaceFlinger 合成可以并行
- 通信方式：mmap 共享内存（零拷贝，不经过 Binder）

**Window 类型与层级（Z-order）：**

```
TYPE_APPLICATION (1-99)            — 普通 Activity 窗口
TYPE_APPLICATION_STARTING (3)      — 启动画面
TYPE_APPLICATION_OVERLAY (2038)    — 悬浮窗
TYPE_STATUS_BAR (2000)             — 状态栏
TYPE_TOAST (2005)                  — Toast
TYPE_SYSTEM_ALERT (2003)           — 系统弹窗
// Z-order 越大越靠前
```

### 实战层（项目经验）

- **悬浮窗实现**：需要 `SYSTEM_ALERT_WINDOW` 权限，通过 `WindowManager.addView()` 添加 TYPE_APPLICATION_OVERLAY 类型窗口
- **Dialog 与 Activity 的关系**：Dialog 有自己的 Window 但共享 Activity 的 WindowToken
- **PopupWindow**：不是真正的 Window，是通过 WindowManager.addView 添加的子窗口
- **多窗口/分屏**：Android 7.0+ 引入，WMS 通过 TaskStack 管理分屏布局
- **性能关联**：过多的 Window 层级会增加 SurfaceFlinger 合成负担

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| WMS 和 AMS 的关系？ | 都运行在 system_server，AMS 管 Activity/进程，WMS 管窗口/Surface |
| Surface 是谁创建的？ | WMS 在 relayoutWindow 中分配，App 通过 ViewRootImpl 持有引用 |
| WMS 参与 View 绘制吗？ | 不参与 measure/layout/draw，只负责 Surface 分配和窗口管理 |
| 输入事件怎么找到目标窗口？ | InputManagerService 根据触摸坐标 + WMS 的 Z-order 确定目标窗口 |
| 为什么 AlertDialog 用 baseContext 崩溃？ | baseContext 没有 Activity 的 Window Token，WMS 校验失败 |

### 延伸问题

- [[Activity显示原理]] — 从 Activity 到屏幕像素的完整链路
- [[View绘制三大流程]] — measure/layout/draw 的详细源码
- [[事件分发机制]] — 输入事件从 WMS 到 View 的完整分发链
- [[Activity启动流程]] — AMS 调度与 WMS 窗口创建的关联
- [[SurfaceView vs TextureView]] — 特殊的 Surface 使用方式

## 记忆锚点

WMS 是窗口的"物业管理"——管谁住在哪层（Z-order）、多大面积（布局）、谁来敲门（输入事件分发）、画板分配（Surface）。但 WMS 不管住户怎么装修（View 绘制是 App 自己的事），也不管画板上画了什么（Surface 内容由 App 直接写入 BufferQueue 给 SurfaceFlinger）。
