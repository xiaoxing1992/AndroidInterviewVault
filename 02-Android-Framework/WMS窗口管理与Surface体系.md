---
tags: [android, 面试, framework, wms, S]
difficulty: S
frequency: medium
---

# WMS 窗口管理与 Surface 体系

## 问题

> WMS（WindowManagerService）的职责是什么？Window、Surface、SurfaceFlinger 之间的关系是怎样的？一个 Activity 的显示涉及哪些层次？

## 核心答案

WMS 负责窗口的创建、布局、层级管理和输入事件分发。每个 Window 对应一个 Surface（绘图画布），App 在 Surface 的 Buffer 上绘制内容，SurfaceFlinger 负责将所有 Surface 合成最终画面输出到屏幕。层次关系：Activity → PhoneWindow → DecorView → ViewRootImpl → Surface → SurfaceFlinger。

## 深入解析

### 原理层

**显示体系层次：**
```
应用层:    Activity → PhoneWindow → DecorView (View 树根)
框架层:    ViewRootImpl → WindowManager → WMS (system_server)
Native层:  Surface → BufferQueue → SurfaceFlinger → HWC → Display
```

**Window 类型与层级（Z-order）：**
```
TYPE_APPLICATION (1-99)        — 普通 Activity 窗口
TYPE_APPLICATION_OVERLAY (2038) — 悬浮窗
TYPE_STATUS_BAR (2000)         — 状态栏
TYPE_TOAST (2005)              — Toast
TYPE_SYSTEM_ALERT (2003)       — 系统弹窗
// Z-order 越大越靠前
```

**Surface 与 BufferQueue：**
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

**WMS 核心职责：**
1. **窗口管理**：addWindow / removeWindow / relayoutWindow
2. **布局计算**：确定每个窗口的位置和大小
3. **Z-order 管理**：决定窗口的前后层级
4. **动画管理**：窗口切换动画
5. **输入事件分发**：确定触摸事件发给哪个窗口

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

### 实战层

- **悬浮窗实现**：需要 `SYSTEM_ALERT_WINDOW` 权限，通过 `WindowManager.addView()` 添加 TYPE_APPLICATION_OVERLAY 类型窗口
- **Dialog 与 Activity 的关系**：Dialog 有自己的 Window 但共享 Activity 的 WindowToken
- **PopupWindow**：不是真正的 Window，是通过 WindowManager.addView 添加的子窗口
- **多窗口/分屏**：Android 7.0+ 引入，WMS 通过 TaskStack 管理分屏布局
- **性能关联**：过多的 Window 层级会增加 SurfaceFlinger 合成负担

### 延伸问题

- [[View绘制三大流程]]
- [[SurfaceView vs TextureView]]
- [[Activity启动流程]]

## 记忆锚点

WMS 是窗口的物业管理（谁住哪层、多大面积），Surface 是每户的画板，SurfaceFlinger 是最终把所有画板叠在一起投影到屏幕的投影仪。
