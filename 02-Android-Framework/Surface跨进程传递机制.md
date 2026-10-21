---
tags: [android, 面试, framework, surface, binder, parcelable, mmap, S]
difficulty: S
frequency: high
---

# Surface 跨进程传递机制

## 问题

> Surface 是怎么跨进程传递的？传递的是像素数据吗？WMS 在其中扮演什么角色？

## 核心答案

Surface 跨进程传递靠 **Parcelable + Binder + native_handle**。Surface 实现了 Parcelable 接口，序列化时只传递一个指向 SurfaceFlinger 侧 GraphicBufferProducer 的 Binder 句柄（BINDER_TYPE_HANDLE）和 native_handle（包含 fd），不是像素数据。真正的像素数据通过 BufferQueue 的 mmap 共享内存跨进程，App 和 SurfaceFlinger 读写同一块物理内存，零拷贝。WMS 的角色是创建 SurfaceControl、请求 SurfaceFlinger 分配 Layer 和 BufferQueue，然后把 Surface 句柄传给 App。

## 深入解析

### 原理层（源码级）

**传递链路全景：**

```
App 进程 (请求方)              system_server (WMS)              SurfaceFlinger
───────────────              ──────────────────              ────────────────
relayoutWindow()
  → 请求 Surface       ──Binder──→  WMS.relayoutWindow()
                                      │
                                      ├─ 创建 SurfaceControl
                                      ├─ 请求 SurfaceFlinger 创建 Layer
                                      └─ 返回 Surface (Parcelable) 给 App
                                ←──Binder── Surface (序列化 Binder 句柄 + fd)
  ← 收到 Surface

Surface 内部：
  只是一个"句柄"
  → 持有 IGraphicBufferProducer 的 Binder 代理
  → 真正的 Buffer 在 SurfaceFlinger 侧管理
  → 通过 mmap 共享内存传递像素数据
```

---

#### Step 1：App 请求 Surface

```java
// ViewRootImpl.java
private int relayoutWindow(WindowManager.LayoutParams params, ...) {
    // 通过 Binder 调用 WMS
    int relayoutResult = mWindowSession.relayout(
        mWindow,           // IWindow（App 侧的 Binder 接口）
        params,            // 窗口参数
        requestedWidth,
        requestedHeight,
        viewVisibility,
        mTmpFrames,        // 输出：窗口帧信息
        mPendingMergedConfiguration,
        mSurfaceControl,   // ★ 输出：SurfaceControl（包含 Surface 信息）
        ...);
    return relayoutResult;
}
```

---

#### Step 2：WMS 创建 Surface

```java
// WindowManagerService.java → WindowState.java
int relayoutWindow(Session session, IWindow client, ...) {
    // 创建或更新 WindowSurfaceController
    result = createSurfaceControl(outSurfaceControl, result, win, ...);
    // → WindowSurfaceController 创建 SurfaceControl
    // → SurfaceControl.nativeCreate() 在 native 层创建
    // → 底层调用 SurfaceComposerClient::createSurface()
    // → SurfaceFlinger 侧创建真正的 Layer 和 BufferQueue
}
```

WMS 在 Surface 传递中的角色：

```
WMS 做了什么：
  ① 创建 SurfaceControl（窗口表面的控制句柄）
  ② 通过 SurfaceFlinger 的 Binder 接口请求创建 Layer
  ③ SurfaceFlinger 创建 Layer + BufferQueue
  ④ WMS 把 SurfaceControl 序列化到 Parcel 返回给 App
  ⑤ App 从 Parcel 还原 Surface，获得 GraphicBufferProducer 的 Binder 代理
```

---

#### Step 3：Surface 的 Parcelable 序列化

```java
// Surface.java
public final class Surface implements Parcelable {

    // ★ 序列化：把 native handle 写入 Parcel
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        nativeWriteToParcel(mNativeObject, dest);
    }

    // ★ 反序列化：从 Parcel 重建 Surface
    public static final Creator<Surface> CREATOR = new Creator<Surface>() {
        @Override
        public Surface createFromParcel(Parcel source) {
            return new Surface(source, false);
        }
    };

    private Surface(Parcel in) {
        mNativeObject = nativeReadFromParcel(in);
    }
}
```

```cpp
// android_view_Surface.cpp
static jlong nativeReadFromParcel(JNIEnv* env, jclass clazz, jobject parcelObj) {
    Parcel* parcel = parcelForJavaObject(env, parcelObj);

    // 从 Parcel 中读取 flat_binder_object
    // 类型是 BINDER_TYPE_HANDLE
    // 指向 SurfaceFlinger 侧的 GraphicBufferProducer

    sp<Surface> surface = Surface::readFromParcel(parcel);
    // → 内部通过 IGraphicBufferProducer::asInterface()
    //   → 把 BINDER_TYPE_HANDLE 转为 BpGraphicBufferProducer（Binder 代理）

    return reinterpret_cast<jlong>(surface.get());
}
```

---

#### Step 4：Surface 内部结构

```
Surface 对象（App 进程）
  │
  ├── mNativeObject (native 层指针)
  │     │
  │     └── android::Surface
  │           │
  │           ├── mGraphicBufferProducer (IGraphicBufferProducer)
  │           │     │
  │           │     └── BpGraphicBufferProducer
  │           │           │
  │           │           └── Binder 代理 → 指向 SurfaceFlinger 侧的 BufferQueueProducer
  │           │
  │           └── mSlots[3] (本地缓存的 GraphicBuffer)
  │                 │
  │                 └── GraphicBuffer
  │                       │
  │                       └── 通过 mmap 映射的共享内存（真正的像素数据）
  │
  └── 方法：lockCanvas() / unlockCanvasAndPost() / ...
```

---

#### Step 5：Parcel 中传递的是什么？

```
Surface 跨进程传递时，序列化的是什么？

不是像素数据，不是整个 Surface 对象
而是一个 Binder 句柄 + native_handle

Parcel 中的数据：
  ├── flat_binder_object
  │     ├── type = BINDER_TYPE_HANDLE
  │     ├── handle = xxx（Binder 驱动分配的句柄）
  │     └── 指向 SurfaceFlinger 侧的 GraphicBufferProducer
  └── native_handle
        ├── numFds = N
        ├── numInts = M
        └── data[] = {fd1, fd2, ..., int1, int2, ...}
              └── fd 指向 GraphicBuffer 的 dmabuf/ion 内存

接收方收到 Parcel 后：
  → 通过 BINDER_TYPE_HANDLE 还原 BpGraphicBufferProducer（Binder 代理）
  → 通过 fd 还原 GraphicBuffer 的 mmap 映射
  → 现在可以 dequeueBuffer / queueBuffer
```

---

### Buffer 的跨进程传递（mmap 共享内存）

```
Surface 的 Binder 代理只是"通道"
真正的像素数据通过 BufferQueue 的 mmap 共享内存传递

App 进程 (Producer)              内核空间                 SurfaceFlinger (Consumer)
───────────────                ──────────               ────────────────────────
lockCanvas()
  → dequeueBuffer()
    → 通过 Binder 代理请求 SurfaceFlinger
    → SurfaceFlinger 返回 GraphicBuffer 的 fd
    → mmap(fd) 映射到 App 进程地址空间
    → App 和 SurfaceFlinger 都能访问同一块物理内存

  → Canvas 绘制到这块共享内存

unlockCanvasAndPost()
  → queueBuffer()
    → 通过 Binder 通知 SurfaceFlinger "Buffer 已填充"
    → SurfaceFlinger 直接从共享内存读取（零拷贝）
    → 合成所有 Layer → 送显
```

**为什么不用 Binder 传像素数据？**

```
① 大小限制：Binder 事务上限约 1MB，一帧 1080p 约 8MB
② 拷贝开销：Binder 传数据需要 1 次拷贝（copy_from_user）
③ 频率高：60fps 每秒 60 帧，拷贝开销不可接受

mmap 共享内存的优势：
  → 零拷贝：两个进程直接读写同一块物理内存
  → 无大小限制：受物理内存限制
  → 高频友好：只有 fd 传递经过 Binder（极小），像素数据走 mmap
```

---

### native_handle 是什么？

```
native_handle 是 Android native 层的跨进程句柄：

native_handle_t {
    int version;     // 版本
    int numFds;      // 文件描述符数量
    int numInts;     // 整数数量
    int data[0];     // 变长数组：先放 fd，再放 int
}

Surface 序列化时：
  → 把 GraphicBuffer 的 fd（dmabuf/ion）打包到 native_handle
  → 把 Binder handle（指向 BufferQueueProducer）打包到 flat_binder_object
  → 写入 Parcel

Surface 反序列化时：
  → 从 Parcel 中读取 native_handle
  → 通过 fd 重建 GraphicBuffer 的 mmap 映射
  → 通过 Binder handle 重建 BpGraphicBufferProducer
```

**Binder 驱动如何传递 fd？**

```
Binder 驱动在传递 fd 时：
  → 自动把 fd 翻译为目标进程中的新 fd
  → 两个进程的 fd 值不同，但指向同一个内核对象（dmabuf/ion）

传递过程：
  App 进程 fd=25 ──Binder 驱动──→ WMS 进程 fd=42
  fd 值不同，但都指向同一块 GraphicBuffer 内核对象
```

---

### 完整跨进程传递图

```
App 进程                    Binder 驱动               system_server (WMS)
──────────                 ─────────────              ──────────────────

relayoutWindow()
  → 请求 Surface ──Binder──→  翻译 Binder 对象
                                    │
                              WMS.relayoutWindow()
                                → 创建 SurfaceControl
                                → SurfaceFlinger 创建 Layer + BufferQueue
                                → 序列化 Surface (Parcelable)
                                  → writeToParcel()
                                    → nativeWriteToParcel()
                                      → 写入 flat_binder_object (BINDER_TYPE_HANDLE)
                                      → 写入 native_handle (包含 fd)
                                    │
                              ←──Binder──  传递 Parcel
                                    │
                              Binder 驱动翻译 fd 和 handle
                                    │
App 侧收到 Parcel
  → Surface(Parcel in)
    → nativeReadFromParcel()
      → 从 flat_binder_object 还原 BpGraphicBufferProducer
      → 从 native_handle 还原 GraphicBuffer 的 mmap 映射
  → 现在 App 持有可用的 Surface
    → lockCanvas() → dequeueBuffer() → 绘制 → queueBuffer()
```

---

### 实战层（项目经验）

**Surface 的跨进程传递总结：**

```
传递方式              传了什么                  大小        频率
──────────           ──────────               ──────     ──────
Binder (Parcel)      Binder 句柄 + fd          ~几百字节   一次性
mmap 共享内存         像素数据 (GraphicBuffer)   ~8MB/帧    每帧
Binder (BufferQueue)  dequeue/queue 命令        ~几十字节   每帧
```

**SurfaceView 和普通 View 的 Surface 区别：**

```
普通 View：
  → 共享 Activity Window 的 Surface
  → 所有 View 绘制到同一个 Surface 的同一个 Buffer
  → Surface 由 ViewRootImpl 管理

SurfaceView：
  → 有自己独立的 Surface
  → 不走 ViewRootImpl 的绘制流程
  → 可以在子线程直接绘制（独立于主线程）
  → Surface 由 SurfaceView 内部创建
  → 适合视频播放、相机预览、游戏等高频渲染场景
```

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| Surface 跨进程传递时传的是什么？ | Binder 句柄 + native_handle（fd），不是像素数据 |
| 像素数据怎么跨进程？ | BufferQueue 的 mmap 共享内存，零拷贝 |
| Surface 实现了什么接口？ | Parcelable，可序列化到 Parcel |
| Binder 怎么传递 fd？ | Binder 驱动自动翻译 fd，两个进程 fd 值不同但指向同一内核对象 |
| 为什么不直接用 Binder 传像素？ | Binder 上限 1MB，一帧 1080p 约 8MB，且拷贝开销大 |
| GraphicBufferProducer 是什么？ | BufferQueue 的生产者端接口，App 通过它 dequeueBuffer/queueBuffer |
| WMS 在 Surface 传递中的角色？ | 创建 SurfaceControl，请求 SurfaceFlinger 分配 Layer，把 Surface 句柄传给 App |
| SurfaceView 和普通 View 的 Surface 区别？ | SurfaceView 有独立 Surface，可子线程绘制；普通 View 共享 Activity 的 Surface |

### 延伸问题

- [[WMS窗口管理与Surface体系]] — WMS 窗口管理与 Surface 体系
- [[Binder通信原理]] — Binder 一次拷贝原理
- [[Activity显示原理]] — Activity 到屏幕的完整显示链路
- [[SurfaceView vs TextureView]] — 两种特殊 Surface 使用方式

## 记忆锚点

Surface 跨进程传的不是"画"而是"画笔的柄"。Parcelable 序列化一个 Binder 句柄（指向 SurfaceFlinger 的 BufferQueueProducer）+ fd（指向 mmap 共享内存）。WMS 是中间人，帮 App 向 SurfaceFlinger 要一块画布（Layer + BufferQueue），再把画笔柄递给 App。像素数据走 mmap 零拷贝，Binder 只传控制命令。
