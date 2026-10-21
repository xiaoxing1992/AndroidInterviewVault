---
tags: [android, 面试, framework, surface, bufferqueue, gpu, rendering, S]
difficulty: S
frequency: high
---

# Surface 绘制原理

## 问题

> Surface 的绘制原理是什么？lockCanvas/unlockCanvasAndPost 做了什么？像素数据是怎么从 Canvas 到屏幕的？BufferQueue 的工作机制是怎样的？

## 核心答案

Surface 是生产者端接口，通过 IGraphicBufferProducer（Binder 代理）与 SurfaceFlinger 侧的 BufferQueue 通信。lockCanvas 做三件事：dequeueBuffer 从 BufferQueue 获取空闲 Buffer → mmap 映射到当前进程 → 返回 Canvas（持有内存地址）。Canvas.drawXXX() 直接写入 mmap 共享内存（进程内操作）。unlockCanvasAndPost 做两件事：unlock 释放 CPU 锁定 → queueBuffer 提交 Buffer 到 BufferQueue。SurfaceFlinger 在 VSYNC-sf 时 acquireBuffer 取走 Buffer 合成送显。硬件加速下不走 lockCanvas，而是主线程构建 DisplayList → RenderThread → GPU 渲染到 Buffer → eglSwapBuffers 提交。

## 深入解析

### 原理层（源码级）

**Surface 的本质：**

```
Surface 不是"画布"，而是"画笔的柄"

Surface 类结构：
  Surface
    ├── mNativeObject → android::Surface (native)
    │     ├── mGraphicBufferProducer  ← Binder 代理，指向 SF 的 BufferQueueProducer
    │     ├── mSlots[3]              ← 本地缓存的 GraphicBuffer（Triple Buffering）
    │     └── mWidth / mHeight / mFormat
    │
    ├── lockCanvas(Rect dirty)       ← 获取 Canvas，锁定 Buffer
    ├── unlockCanvasAndPost(Canvas)  ← 提交 Buffer 到 BufferQueue
    └── dequeueBuffer / queueBuffer  ← 底层 Buffer 操作

Surface 是 App 进程与 SurfaceFlinger 之间的桥梁：
  App 侧：通过 Surface 的 Binder 代理发送 dequeue/queue 命令
  SF 侧：BufferQueue 管理 Buffer 的分配和回收
  像素数据：通过 mmap 共享内存传递，零拷贝
```

---

#### lockCanvas 完整流程

```java
// Surface.java
public Canvas lockCanvas(Rect dirty) {
    return lockCanvasNative(dirty);
}
```

```cpp
// android_view_Surface.cpp
static jobject nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {

    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    ANativeWindow_Buffer buffer;

    // ★ 关键：dequeueBuffer + lock
    status_t err = surface->lock(&buffer, dirtyRect);

    // Canvas 内部持有这块 Buffer 的内存地址
    // 后续 Canvas.drawXXX() 直接写入这块内存
    return canvasObj;
}
```

```cpp
// Surface.cpp
status_t Surface::lock(ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds) {
    // ① 从 BufferQueue dequeue 一块空闲 Buffer
    ANativeWindowBuffer* anb;
    int slot;
    status_t err = dequeueBuffer(&anb, &slot);
    // → 通过 Binder 代理调用 SurfaceFlinger 的 BufferQueueProducer.dequeueBuffer()
    // → SurfaceFlinger 从 BufferQueue 中取出一块空闲 Buffer
    // → 返回 Buffer 的 fd

    // ② mmap 映射到当前进程
    sp<GraphicBuffer> graphicBuffer = mSlots[slot].buffer;
    void* vaddr;
    status_t err2 = graphicBuffer->lock(GRANT_READ_WRITE, &vaddr);
    // → mmap(fd, size) → 把 Buffer 映射到 App 的用户空间
    // → vaddr 就是这块 Buffer 的内存地址

    // ③ 设置输出参数
    outBuffer->width  = graphicBuffer->getWidth();
    outBuffer->height = graphicBuffer->getHeight();
    outBuffer->stride = graphicBuffer->getStride();
    outBuffer->format = graphicBuffer->getFormat();
    outBuffer->bits   = vaddr;  // ★ 像素数据的内存地址

    return NO_ERROR;
}
```

```
lockCanvas 做了什么：
  ① dequeueBuffer — 从 BufferQueue 获取空闲 Buffer
  ② mmap — 把 Buffer 映射到 App 进程地址空间
  ③ 返回 Canvas — Canvas 内部持有这块内存地址
  ④ 后续 Canvas.drawXXX() 直接写入这块内存（像素级操作）
```

---

#### Canvas 绘制到 Buffer

```
lockCanvas 返回后，Canvas 内部持有一块内存地址（vaddr）

Canvas.drawRect(rect, paint)
  → Skia 库（2D 渲染引擎）
    → 计算像素值
    → 直接写入 vaddr 指向的内存

Canvas.drawText(text, x, y, paint)
  → Skia 字体渲染
    → 光栅化文字为像素
    → 写入 vaddr

Canvas.drawBitmap(bitmap, src, dst, paint)
  → 像素拷贝/缩放
    → 写入 vaddr

所有 Canvas 操作最终都是对这块共享内存的读写
不经过 Binder，不经过 SurfaceFlinger
纯粹的进程内内存操作
```

---

#### unlockCanvasAndPost 完整流程

```cpp
// Surface.cpp
status_t Surface::unlockAndPost() {
    // ① unlock：释放 CPU 对 Buffer 的锁定
    status_t err = mLockedBuffer->unlock();

    // ② queueBuffer：把已填充的 Buffer 提交回 BufferQueue
    err = queueBuffer(mLockedBuffer.get());

    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = nullptr;
    return err;
}

status_t Surface::queueBuffer(ANativeWindowBuffer* buffer) {
    // 通过 Binder 代理调用 SurfaceFlinger 的 BufferQueueProducer.queueBuffer()
    // 传递：Buffer 的引用、时间戳、脏区域、Fence（GPU 渲染完成的同步信号）

    IGraphicBufferProducer::QueueBufferOutput output;
    status_t err = mGraphicBufferProducer->queueBuffer(slot, item, &output);
    // → SurfaceFlinger 收到后，Buffer 进入 BufferQueue 的队列
    // → 等待下一个 VSYNC-sf 时被 SurfaceFlinger 取走合成
}
```

```
unlockCanvasAndPost 做了什么：
  ① unlock — 释放 CPU 对 Buffer 的锁定
  ② queueBuffer — 通过 Binder 代理把 Buffer 提交到 BufferQueue
  ③ SurfaceFlinger 在下一个 VSYNC-sf 取走 Buffer 合成
```

---

### BufferQueue 完整生命周期

```
App (Producer)              BufferQueue              SurfaceFlinger (Consumer)
─────────────              ───────────              ────────────────────────

初始化：
  Surface 创建
    → BufferQueue 有 3 个 Buffer 槽位（Triple Buffering）
    → 初始状态：所有 Buffer 都是 FREE

绘制一帧：

  ① dequeueBuffer()
     → App 请求一块空闲 Buffer
     → BufferQueue 从 FREE 列表中取出一块
     → Buffer 状态：FREE → DEQUEUED
     → App 获得 Buffer 的 fd → mmap 映射
                   │
                   ↓
  ② 绘制
     → Canvas.drawXXX() / GPU 渲染
     → 写入 mmap 映射的共享内存
     → 像素数据直接在共享内存中，不经过 Binder
                   │
                   ↓
  ③ queueBuffer()
     → App 提交已填充的 Buffer
     → Buffer 状态：DEQUEUED → QUEUED
     → 通知 SurfaceFlinger 有新 Buffer
                   │
                   ↓
  ④ acquireBuffer()          ← SurfaceFlinger 在 VSYNC-sf 时调用
     → SurfaceFlinger 从队列中取出 Buffer
     → Buffer 状态：QUEUED → ACQUIRED
     → 合成所有 Layer → 送显
                   │
                   ↓
  ⑤ releaseBuffer()          ← SurfaceFlinger 合成完成后
     → Buffer 状态：ACQUIRED → FREE
     → Buffer 回到空闲池，等待下一次 dequeueBuffer
```

```
Buffer 状态机：
  FREE ──dequeue──→ DEQUEUED ──queue──→ QUEUED ──acquire──→ ACQUIRED ──release──→ FREE

Triple Buffering 下 3 个 Buffer 轮转：
  Buffer A: FREE（等待 dequeue）
  Buffer B: QUEUED（等待 SurfaceFlinger acquire）
  Buffer C: ACQUIRED（SurfaceFlinger 正在合成/显示）
  → 三者并行，互不阻塞
```

---

### Fence 同步机制

```
GPU 渲染是异步的，CPU 提交 queueBuffer 时 GPU 可能还没渲染完

Fence（栅栏）解决这个问题：
  → GPU 渲染完成后发出 signal
  → SurfaceFlinger acquireBuffer 时检查 Fence
  → Fence 已 signal → 可以安全读取 Buffer
  → Fence 未 signal → 等待 GPU 渲染完成

Fence 类型：
  → Acquire Fence：App 发给 SurfaceFlinger，表示"GPU 渲染完成，可以读了"
  → Release Fence：SurfaceFlinger 发给 App，表示"显示完了，可以写了"
```

```
时序：
  App (CPU):     dequeueBuffer → 绘制命令 → queueBuffer
  App (GPU):                    渲染 → signal acquireFence
  SurfaceFlinger:                           acquireBuffer（等 acquireFence）
                                            合成 → signal releaseFence
  App:                                        dequeueBuffer（等 releaseFence）
```

---

### 软件渲染 vs 硬件加速

```
软件渲染路径（lockCanvas）：
  Surface.lockCanvas()
    → dequeueBuffer → mmap → 返回 Canvas
  Canvas.drawXXX()
    → CPU 直接写入共享内存（Skia 光栅化）
  Surface.unlockCanvasAndPost()
    → queueBuffer → 提交给 SurfaceFlinger

  特点：CPU 完成所有渲染，简单但慢

硬件加速路径（ThreadedRenderer，API 14+ 默认开启）：
  主线程：
    View.updateDisplayListIfDirty()
      → RecordingCanvas 记录绘制命令到 DisplayList
      → 不直接写入 Buffer
  主线程 → RenderThread：
    syncAndDrawFrame()
      → RenderThread 遍历 DisplayList
      → 转换为 OpenGL/Vulkan 指令
      → GPU 渲染到 Buffer（eglSwapBuffers）
        → 内部 dequeueBuffer → GPU 渲染 → queueBuffer

  特点：GPU 渲染，性能高，主线程不等 GPU 完成
```

---

### 完整时序图

```
App 主线程                 RenderThread              BufferQueue              SurfaceFlinger
──────────                ─────────────             ───────────              ──────────────

invalidate()
  → scheduleTraversals()
    → 等 VSYNC
VSYNC 到来
  → performTraversals()
    → measure/layout
    → draw
      → 构建 DisplayList
      → syncAndDrawFrame()
                                ← 收到 DisplayList
                                遍历 DisplayList
                                转为 GPU 指令
                                GPU 渲染
                                eglSwapBuffers()
                                  → dequeueBuffer() ──→ 取出 FREE Buffer
                                  → GPU 写入 Buffer
                                  → queueBuffer()   ──→ Buffer 入队
                                                                → VSYNC-sf
                                                                acquireBuffer() ──→ 取出 Buffer
                                                                合成 Layer
                                                                HWC → Display
                                                                releaseBuffer() ──→ Buffer 回收
```

---

### 实战层（项目经验）

**dequeueBuffer 可能阻塞：**

```
如果所有 Buffer 都在使用中（SF 还没 release）：
  → dequeueBuffer 会阻塞等待
  → 主线程（软件渲染）或 RenderThread（硬件加速）被阻塞
  → 可能导致掉帧

Triple Buffering 缓解了这个问题：
  → 3 个 Buffer 轮转，通常总有一个是 FREE 的
  → 只有在 GPU 负载极高时才会出现所有 Buffer 都被占用
```

**SurfaceView vs 普通 View 的 Surface：**

```
普通 View：
  → 共享 Activity Window 的 Surface
  → 所有 View 绘制到同一个 Surface 的同一个 Buffer
  → Surface 由 ViewRootImpl 管理

SurfaceView：
  → 有自己独立的 Surface（独立的 BufferQueue）
  → 不走 ViewRootImpl 的绘制流程
  → 可以在子线程直接绘制（独立于主线程）
  → 适合视频播放、相机预览、游戏等高频渲染场景
```

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| Surface 是什么？ | 不是画布，是生产者端接口，通过 Binder 代理与 SurfaceFlinger 的 BufferQueue 通信 |
| lockCanvas 做了什么？ | dequeueBuffer 获取空闲 Buffer → mmap 映射到当前进程 → 返回 Canvas（持有内存地址） |
| unlockCanvasAndPost 做了什么？ | unlock 释放 CPU 锁定 → queueBuffer 提交 Buffer 到 BufferQueue |
| 像素数据怎么到屏幕的？ | Canvas 写入 mmap 共享内存 → queueBuffer → SurfaceFlinger acquireBuffer → 合成 → Display |
| Triple Buffering 的作用？ | 3 个 Buffer 轮转，App 绘制/SF 合成/Display 显示三者并行 |
| Fence 是什么？ | GPU 渲染完成的同步信号，保证 SurfaceFlinger 不会在 GPU 还没渲染完就读 Buffer |
| 软件渲染和硬件加速的区别？ | 软件：lockCanvas → CPU 光栅化。硬件：DisplayList → RenderThread → GPU 渲染 |
| dequeueBuffer 可能阻塞吗？ | 会，如果所有 Buffer 都在使用中，会等待 SurfaceFlinger release |

### 延伸问题

- [[Surface跨进程传递机制]] — Surface 如何通过 Parcelable + Binder 传递
- [[WMS窗口管理与Surface体系]] — WMS 如何分配 Surface
- [[View绘制三大流程]] — View 树的 measure/layout/draw
- [[SurfaceView vs TextureView]] — 两种特殊 Surface 使用方式
- [[Activity显示原理]] — Activity 到屏幕的完整显示链路
- [[丢帧原因与检测方案详解]] — dequeueBuffer 阻塞导致的掉帧

## 记忆锚点

Surface 绘制三步走：lockCanvas（拿 Buffer 画布）→ Canvas.drawXXX（写像素）→ unlockCanvasAndPost（交作业）。lockCanvas = dequeueBuffer + mmap，unlockCanvasAndPost = unlock + queueBuffer。像素数据走 mmap 共享内存零拷贝，Binder 只传 dequeue/queue 控制命令。BufferQueue 四态轮转：FREE → DEQUEUED → QUEUED → ACQUIRED → FREE。
