---
tags: [android, 面试, framework, ipc, binder, socket, mmap, S]
difficulty: S
frequency: high
---

# Android Framework 中的 IPC 方式

## 问题

> Android Framework 中用到了哪些 IPC 方式？各自用在什么场景？为什么不同场景用不同的 IPC？

## 核心答案

Android Framework 主要用 7 种 IPC：① Binder（90%+，App↔系统服务）② Socket（Zygote fork，因为 fork 要求单线程）③ Pipe/eventfd（MessageQueue 线程唤醒）④ mmap 共享内存（BufferQueue 图形数据，零拷贝）⑤ Signal（进程控制 SIGKILL/SIGQUIT）⑥ HIDL/AIDL（HAL 层接口，底层仍是 Binder）⑦ BitTube（VSYNC 信号，socket pair 轻量高频）。选择原则：高频大数据用 mmap，高频小数据用 BitTube，标准系统调用用 Binder，fork 用 Socket，线程唤醒用 eventfd。

## 深入解析

### 原理层（源码级）

#### ① Binder — Android 最核心的 IPC

```
占 Android 跨进程通信的 90% 以上

原理：基于 mmap 的一次拷贝（传统 IPC 需要两次）
特点：C/S 架构、内核校验 UID/PID（安全性高）、面向对象
```

```
Framework 中的 Binder 使用场景：

  App ↔ AMS          startActivity / bindService / broadcastIntent
  App ↔ WMS          addWindow / relayoutWindow
  App ↔ PMS          getPackageInfo / resolveActivity
  App ↔ SurfaceFlinger  dequeueBuffer / queueBuffer（控制命令）
  SystemService 之间   大量互调用
```

```java
// 服务端（system_server）
class ActivityManagerService extends IActivityManager.Stub {
    @Override
    public int startActivity(IApplicationThread caller, Intent intent, ...) {
        // 处理请求
    }
}

// 客户端（App 进程）
IActivityManager am = ActivityManager.getService(); // 获取 Binder 代理
am.startActivity(...); // 跨进程调用
```

```
Binder 线程池模型：
  → 默认最大 16 个线程
  → BinderThread-1 处理请求 A
  → BinderThread-2 处理请求 B
  → 并行处理，不阻塞主线程

  发起方线程阻塞等待（同步调用）
  接收方从线程池取空闲线程处理
  处理完后唤醒发起方线程
```

---

#### ② Socket — Zygote fork 通信

```
使用场景：AMS → Zygote 请求 fork 进程

为什么不用 Binder？
  → fork 要求单线程环境
  → Binder 是多线程模型（Binder 线程池）
  → fork 会复制当前进程，其他线程消失
  → 如果其他线程持有锁 → 子进程死锁
  → Socket 是单线程安全的
```

```java
// ZygoteProcess.java — AMS 侧
public Process.ProcessStartResult start(...) {
    return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), args);
}

private Process.ProcessStartResult zygoteSendArgsAndGetResult(
        ZygoteState zygoteState, List<String> args) {
    final BufferedWriter writer = zygoteState.mZygoteOutputWriter;
    writer.write(Integer.toString(args.size()));
    for (String arg : args) {
        writer.newLine();
        writer.write(arg);
    }
    writer.flush();
    int pid = zygoteState.mZygoteInputStream.readInt();
}
```

```
Zygote 侧：
  ZygoteServer.runSelectLoop()
    → 监听 socket 连接
    → 收到请求 → forkAndSpecialize()
    → 子进程进入 ActivityThread.main()
    → 父进程继续等待下一个请求
```

---

#### ③ Pipe / eventfd — 线程唤醒

```
使用场景：
  → MessageQueue 的 epoll 机制
  → nativeWake() 唤醒 epoll_wait 阻塞的线程
  → Handler 消息循环的唤醒
```

```cpp
// Looper.cpp
Looper::Looper() {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    // eventfd 是 Linux 的事件通知机制
    // 比 pipe 更轻量（只需要一个 fd，pipe 需要两个）

    epoll_event event;
    event.events = EPOLLIN;
    event.data.fd = mWakeEventFd;
    epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, &event);
}

void Looper::wake() {
    uint64_t inc = 1;
    write(mWakeEventFd, &inc, sizeof(uint64_t));  // 唤醒 epoll_wait
}

int Looper::pollInner(int timeoutMillis) {
    int eventCount = epoll_wait(mEpollFd, events, maxEvents, timeoutMillis);
    // 有事件或超时时返回
}
```

```
eventfd vs pipe：
  eventfd：只传递一个计数器（8 字节），更轻量，一个 fd
  pipe：可以传递任意数据，但需要两个 fd（读端/写端）
  MessageQueue 用 eventfd，只需要唤醒信号，不需要传数据
```

---

#### ④ 共享内存 (mmap) — 图形数据传递

```
使用场景：
  → BufferQueue：App 与 SurfaceFlinger 之间的图形 Buffer
  → Ashmem/SharedMemory：大数据跨进程传递
```

```
BufferQueue 的 mmap：
  App 进程                               SurfaceFlinger
  ────────                               ──────────────
  dequeueBuffer()
    → 通过 Binder 获取 GraphicBuffer 的 fd
    → mmap(fd, size)                     → 同一块物理内存
    → 获得内存地址 vaddr
  Canvas.drawXXX()
    → 直接写入 vaddr（不经过 Binder）
  queueBuffer()
    → 通过 Binder 通知 SF "Buffer 已填充"
                                           → acquireBuffer()
                                           → 直接从共享内存读取
                                           → 零拷贝
```

```
为什么不用 Binder 传图形数据？
  → 1080p 一帧约 8MB，Binder 上限 1MB
  → 60fps 每秒 60 帧，拷贝开销不可接受
  → mmap 共享内存，零拷贝
```

```
Ashmem / SharedMemory：
  → Android 特有的共享内存机制
  → 可以跨进程共享，支持按需分配
  → 内存不足时可以被内核回收
  → 用途：大数据传递（图片、大数组等）

  使用方式：
  MemoryFile → 底层是 Ashmem（旧 API）
  SharedMemory → Android 8.0+ 推荐
```

---

#### ⑤ Signal — 进程信号

```
使用场景：
  → Zygote fork 后子进程收到 SIGCHLD
  → AMS 杀进程：Process.killProcess(pid) → SIGKILL
  → ANR 超时：发送 SIGQUIT（dump 堆栈）
  → 调试器附加：SIGSTOP / SIGCONT
```

```
信号类型：
  SIGKILL   → 强制杀死进程（不可捕获）
  SIGTERM   → 请求进程退出（可捕获）
  SIGCHLD   → 子进程状态变化通知父进程
  SIGQUIT   → 请求进程 dump 堆栈（ANR 时使用）
  SIGSTOP   → 暂停进程（不可捕获）
  SIGCONT   → 恢复进程
```

```java
// Process.java
public static void killProcess(int pid) {
    sendSignal(pid, SIGNAL_KILL);  // 发送 SIGKILL
}

// ANR 时 dump 堆栈
public static void sendSignal(int pid, int signal) {
    nativeSendSignal(pid, signal);
    // → kill(pid, signal)
}
```

---

#### ⑥ HIDL/AIDL — HAL 层接口

```
使用场景：Framework ↔ HAL 层（硬件抽象层）

HIDL（HAL Interface Definition Language）：
  → Android 8.0 引入，用于 HAL 接口定义
  → 底层传输可以用 Binder 或 hwbinder
  → 分离 Framework 和 Vendor（独立更新）

AIDL（Android Interface Definition Language）：
  → Android 13+ 统一用 AIDL 替代 HIDL
  → 更通用，覆盖 Java/C++/NDK
```

```
HIDL/AIDL 的使用场景：

  SurfaceFlinger ↔ HWComposer HAL    → 显示合成
  AudioFlinger ↔ Audio HAL           → 音频播放
  CameraService ↔ Camera HAL         → 相机
  SensorManager ↔ Sensor HAL         → 传感器
  Bluetooth ↔ Bluetooth HAL          → 蓝牙
  WiFi ↔ WiFi HAL                    → WiFi
```

```
HIDL 与 Binder 的关系：
  HIDL 是接口定义语言
  Binder 是底层传输机制
  → HIDL 定义接口 → 生成 C++ 代理/存根 → 通过 Binder 传输
  → 类似 Java AIDL → IInterface.Stub/Proxy → Binder 传输
```

---

#### ⑦ BitTube — VSYNC 信号传递

```
使用场景：SurfaceFlinger → App 进程的 VSYNC 信号传递

本质：socket pair（一对 Unix domain socket）
  → 一个 fd 用于写入（SurfaceFlinger 侧）
  → 一个 fd 用于读取（App 侧）
  → 高效、低延迟
```

```cpp
// BitTube.cpp
class BitTube {
    int mSendFd;
    int mRecvFd;

    BitTube() {
        int fds[2];
        socketpair(AF_UNIX, SOCK_SEQPACKET, 0, fds);
        mSendFd = fds[0];
        mRecvFd = fds[1];
    }

    void sendObjects(Event const* events, int count) {
        write(mSendFd, events, count * sizeof(Event));
    }

    ssize_t recvObjects(Event* events, int count) {
        return read(mRecvFd, events, count * sizeof(Event));
    }
};
```

```
为什么用 BitTube 而不是 Binder？
  → VSYNC 是高频信号（60-120Hz）
  → 每次信号数据量极小（几十字节）
  → socket pair 更轻量，不需要 Binder 的序列化/反序列化
  → 延迟更低
```

---

### 各 IPC 方式对比

```
┌──────────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│     方式      │  拷贝次数  │   延迟    │  数据量   │  安全性   │  使用场景  │
├──────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Binder       │    1     │   中     │  < 1MB   │  高      │ 90%+ IPC │
│ Socket       │    2     │   中     │  不限    │  中      │ Zygote   │
│ Pipe/eventfd │    0     │   低     │  极小    │  低      │ 线程唤醒  │
│ mmap         │    0     │   极低   │  不限    │  中      │ 图形数据  │
│ Signal       │    0     │   极低   │  0       │  低      │ 进程控制  │
│ HIDL/AIDL    │    1     │   中     │  < 1MB   │  高      │ HAL 接口  │
│ BitTube      │    2     │   低     │  极小    │  中      │ VSYNC    │
└──────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

---

### 各场景使用的 IPC

```
场景                              IPC 方式           原因
──────────                       ──────────         ──────────
App ↔ AMS                       Binder             标准系统服务调用
App ↔ WMS                       Binder             窗口管理
App ↔ PMS                       Binder             包管理
App ↔ SurfaceFlinger            Binder             Buffer 控制命令
App ↔ SurfaceFlinger            mmap               图形 Buffer 数据
AMS → Zygote fork               Socket             fork 要求单线程
SF → App VSYNC                  BitTube (socket)   高频小数据
Framework ↔ HAL                 HIDL/AIDL (Binder) 硬件抽象层
MessageQueue 唤醒               eventfd            轻量线程唤醒
AMS 杀进程                      Signal (SIGKILL)   进程控制
ANR dump 堆栈                   Signal (SIGQUIT)   进程诊断
```

---

### 实战层（项目经验）

**选择 IPC 的原则：**

```
高频大数据 → mmap 共享内存（零拷贝）
高频小数据 → BitTube / socket pair（轻量）
标准系统调用 → Binder（安全、通用）
fork 进程 → Socket（单线程安全）
线程唤醒 → eventfd（最轻量）
硬件抽象 → HIDL/AIDL（接口定义 + Binder 传输）
进程控制 → Signal（最底层）
```

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| Android 主要的 IPC 方式是什么？ | Binder，占 90%+ |
| Zygote 为什么用 Socket 不用 Binder？ | fork 要求单线程，Binder 多线程会导致死锁 |
| 图形数据怎么跨进程？ | mmap 共享内存，零拷贝，不经过 Binder |
| VSYNC 信号用什么传递？ | BitTube（socket pair），高频小数据，比 Binder 更轻量 |
| Binder 的优势？ | 一次拷贝、安全性高（内核校验 UID/PID）、C/S 架构 |
| HIDL 和 Binder 的关系？ | HIDL 是接口定义语言，底层传输用 Binder |
| 为什么不用共享内存做所有 IPC？ | 共享内存需要自己处理同步（锁），Binder 由内核保证安全和有序 |
| eventfd 和 pipe 的区别？ | eventfd 只传计数器（8 字节，一个 fd），pipe 传任意数据（两个 fd） |

### 延伸问题

- [[Binder通信原理]] — Binder 一次拷贝原理与架构
- [[Zygote进程启动与App进程孵化]] — Socket 通信与 fork 机制
- [[VSYNC详解]] — BitTube 传递 VSYNC 信号
- [[Surface绘制原理]] — mmap 共享内存传递图形数据
- [[HWComposer详解]] — HIDL 接口与 HAL 层通信

## 记忆锚点

Android IPC 七兄弟：Binder 是老大（干 90% 的活），Socket 管 fork（单线程安全），eventfd 管叫醒（最轻量），mmap 管画画（零拷贝），Signal 管杀进程（最底层），HIDL 管硬件（接口定义），BitTube 管节拍（VSYNC）。选谁看场景：大数据用 mmap，小高频用 Socket，标准调用用 Binder。
