---
tags: [android, 面试, framework, system_server, binder, S]
difficulty: S
frequency: high
---

# SystemServer 启动 Binder 机制详解

## 问题

> SystemServer 中的 Binder 机制是怎么启动的？从内核驱动到服务注册，完整链路是什么？

## 核心答案

SystemServer 的 Binder 启动分三个层次：**内核层**——Binder 驱动在 Linux kernel 启动时加载，创建 `/dev/binder` 设备节点；**进程层**——Zygote fork SystemServer 后，子进程通过 `nativeZygoteInit()` → `ProcessState::self()` 打开 `/dev/binder` 并 mmap 映射缓冲区，然后 `startThreadPool()` 启动 Binder 主线程进入 `joinThreadPool()` 循环等待请求；**服务层**——`ServiceManager.addService()` 将每个服务的 Binder 实体注册到 ServiceManager，之后其他进程通过 `SM.getService()` 获取代理即可发起 IPC 调用。

## 深入解析

### 原理层

**三个层次的时序：**

```text
T1: Linux kernel 启动
    → 加载 Binder 驱动模块
    → 创建 /dev/binder 设备节点

T2: Zygote fork SystemServer 子进程
    → nativeZygoteInit()
      → ProcessState::self()            // 打开 /dev/binder + mmap
      → ProcessState::startThreadPool() // 启动 Binder 线程池

T3: SystemServer.run()
    → ServiceManager.addService("activity", ams)  // 注册服务
    → ServiceManager.addService("window", wms)
    → ...
    → systemReady()  // 服务就绪，开始正常处理请求
```

### 层次一：Binder 驱动加载（内核层）

```text
Linux kernel 启动
  → 内核初始化阶段
    → 加载 Binder 驱动模块（binder_linux.c）
    → 创建 /dev/binder 字符设备
    → 初始化 Binder 驱动内部数据结构：
        ├── binder_procs（所有使用 Binder 的进程链表）
        ├── binder_devices（设备节点）
        ├── 红黑树（Binder 实体和引用的索引）
        └── 等待队列（阻塞的 Binder 线程）
```

Binder 驱动在所有用户态进程之前就加载好了。

### 层次二：SystemServer 的 Binder 线程池初始化

**nativeZygoteInit() 的完整调用链：**

```text
Java 层：
  ZygoteInit.nativeZygoteInit()           // Java native 方法
    ↓ JNI
Native 层：
  com_android_internal_os_ZygoteInit_nativeZygoteInit()
    → gCurRuntime->onZygoteInit()
      → ProcessState::self()               // ★ 第一步：打开 Binder 驱动
        → open("/dev/binder", O_RDWR)       // 获取 fd
        → mmap(1M - 8K)                     // 映射 Binder 缓冲区
      → ProcessState::startThreadPool()     // ★ 第二步：启动 Binder 线程池
        → spawnPooledThread(true)
          → new PoolThread(isMain=true)     // 创建 Binder 主线程
            → IPCThreadState::self()->joinThreadPool()
              → while(true) {
                    talkWithDriver();        // 阻塞在 ioctl
                    executeCommand();        // 处理 Binder 请求
                }
```

**ProcessState —— 进程级 Binder 状态：**

```text
ProcessState::self() 是进程单例：

  open("/dev/binder")
    → 得到 fd（文件描述符）

  mmap(BINDER_VM_SIZE)  // BINDER_VM_SIZE = 1M - 8K = 1040384
    → 在用户态映射一块和内核共享的内存
    → 用于 Binder 数据传输（避免 copy_from_user/copy_to_user）
    → 这块内存 App 进程和 SystemServer 都能访问

  保存 fd 和映射地址到 ProcessState 单例
```

**Binder 线程池的结构：**

```text
SystemServer 进程的线程：

  主线程（main）
    → Looper.loop() 运行 Handler 消息循环
    → systemReady()、服务启动在这里执行

  Binder 主线程（Binder:xxx_1）
    → nativeZygoteInit() 创建
    → joinThreadPool() 循环
    → 等待 → 收到请求 → 调 Stub.onTransact() → 继续等待

  Binder 工作线程（Binder:xxx_2 ~ N）
    → 按需创建（默认最大 15 个）
    → Binder 主线程忙时，驱动自动唤醒或创建新线程
    → 同样 joinThreadPool() 循环

  各服务自己的 Handler 线程
    → android.display、android.ui 等
```

### 层次三：服务注册 Binder 到 ServiceManager

```text
ServiceManager.addService("activity", ams)

  ams 是什么？
    → ActivityManagerService 对象
    → 继承 IActivityManager.Stub
    → Stub 继承 android.os.Binder
    → Binder 对象有唯一标识（Binder 实体 / BBinder）

  注册过程：
    → IServiceManager.addService("activity", ams, ...)
      → BinderProxy.transact(ADD_SERVICE_TRANSACTION, ...)
        → IPCThreadState::transact()
          → ioctl(BINDER_WRITE_READ)
            → Binder 驱动将请求路由到 ServiceManager 进程
              → ServiceManager 的 Binder 线程收到请求
                → svcmgr_handler()
                  → do_add_service()
                    → 记录 "activity" → Binder 实体 映射
```

### 完整的 Binder 通信数据流

```text
App 进程                          Binder 驱动（内核）              SystemServer
─────────                        ─────────────────              ─────────────

ams.startActivity(intent)
  → Proxy.startActivity()
    → Parcel 写入参数
    → mRemote.transact()
      → IPCThreadState::transact()
        → ioctl(BINDER_WRITE_READ)
          → 从 App 缓冲区读数据 ──→
                                  → 复制到 SS 的缓冲区（mmap 共享）
                                  → 唤醒 SS 的 Binder 线程 ──→
                                                            → joinThreadPool()
                                                              → talkWithDriver()
                                                              → executeCommand()
                                                                → Stub.onTransact()
                                                                  → case START_ACTIVITY:
                                                                    → startActivity()
                                                                    → 写 reply
                                                              → ioctl(BINDER_WRITE_READ)
                                  ←── 将 reply 复制回 App ───
  ← waitForResponse() 返回 ←─────
  → 读取 reply，得到结果
```

**mmap 的关键作用：**

```text
传统 IPC（如 socket/pipe）：
  发送方 buffer → copy_from_user() → 内核 buffer → copy_to_user() → 接收方 buffer
  两次内存拷贝

Binder：
  发送方 buffer → copy_from_user() → 内核/接收方共享缓冲区（mmap）
  一次内存拷贝（接收方直接读 mmap 映射区）
```

### 各阶段的 Binder 状态总结

```text
Zygote fork 时：
  → 子进程继承 /dev/binder fd
  → 没有 Binder 线程（fork 只复制主线程）
  → 不能接收任何 Binder 请求

nativeZygoteInit() 之后：
  → /dev/binder 已打开，mmap 已映射
  → Binder 主线程已启动
  → 可以接收 Binder 请求（ServiceManager 可以向它发消息了）

ServiceManager.addService() 之后：
  → 各服务的 Binder 实体已注册
  → 其他进程可以通过 SM 发现这些服务
  → Binder 线程可以接收并分发请求到对应 Stub

systemReady() 之后：
  → 服务进入正常运行状态
  → 开始处理来自 App 的 IPC 调用
```

### 实战层

**面试常见追问：**

- nativeZygoteInit() 是在哪个线程执行的？
- ProcessState 和 IPCThreadState 分别是什么？
- Binder 线程池的最大线程数是多少？由谁控制？
- mmap 映射的缓冲区大小是多少？为什么是这个值？
- Binder 驱动是怎么唤醒 SystemServer 的 Binder 线程的？
- 如果 Binder 线程池满了会怎样？
- SystemServer 的 Binder 线程和主线程是什么关系？

**ProcessState vs IPCThreadState：**

| | ProcessState | IPCThreadState |
|---|---|---|
| 作用 | 进程级：打开 /dev/binder、mmap、管理线程池 | 线程级：发送/接收 Binder 数据 |
| 实例 | 进程单例 | 每个线程一个（ThreadLocal） |
| 职责 | fd 管理、缓冲区管理、线程池管理 | transact()、talkWithDriver()、joinThreadPool() |

**Binder 线程池大小：**

```text
默认最大 15 个工作线程 + 1 个主线程 = 最多 16 个 Binder 线程

由 ProcessState::setThreadPoolMaxThreadCount() 控制
可以通过 sysctl 修改：
  /proc/sys/binder/max_threads
```

**线程池满了会怎样：**

```text
App 调用 SystemServer 的 Binder 方法
  → Binder 驱动尝试唤醒或创建 Binder 线程
  → 线程池已满（15 个都在忙）
  → 请求排队等待（pending transaction）
  → App 端阻塞在 waitForResponse()
  → 超时后 App 收到 TransactionTooLargeException 或 DeadObjectException
```

**排查和观察：**

```bash
# 查看 SystemServer 的线程（含 Binder 线程）
adb shell ps -T -p $(adb shell pidof system_server) | grep -E "main|Binder"

# 查看 Binder 线程数
adb shell cat /proc/$(adb shell pidof system_server)/status | grep Threads

# 查看 Binder 驱动状态
adb shell cat /sys/kernel/debug/binder/stats

# 查看 SystemServer 的 Binder 事务
adb shell cat /sys/kernel/debug/binder/proc/$(adb shell pidof system_server)

# 查看 Binder 缓冲区使用情况
adb shell cat /sys/kernel/debug/binder/stats | grep -A 5 "proc <pid>"
```

**源码入口：**

- `nativeZygoteInit`：`frameworks/base/core/jni/AndroidRuntime.cpp`
- `ProcessState`：`frameworks/native/libs/binder/ProcessState.cpp`
- `IPCThreadState`：`frameworks/native/libs/binder/IPCThreadState.cpp`
- `ServiceManager.addService`：`frameworks/native/libs/binder/IServiceManager.aidl`
- Binder 驱动：`drivers/android/binder.c`

**容易答错的点：**

- Binder 驱动是内核模块，在 SystemServer 之前就加载好了
- `nativeZygoteInit()` 是在 SystemServer 子进程中执行的，不是在 Zygote 中
- Binder 线程池是在子进程 fork 之后才初始化的（fork 后的干净环境中）
- `ProcessState::self()` 只打开一次 /dev/binder，后续所有 Binder 线程共享同一个 fd
- mmap 映射的缓冲区用于减少数据拷贝次数（一次 vs 两次）

### 延伸问题

- [[Binder通信原理]]
- [[Zygote为什么用Socket不用Binder]]
- [[系统服务启动流程]]
- [[ServiceManager与system_server区别]]
- [[SystemServer启动流程]]

## 记忆锚点

三个层次：**内核加载驱动** → **nativeZygoteInit() 打开 /dev/binder + mmap + 启动 Binder 线程池** → **ServiceManager.addService() 注册服务**。Binder 线程池在 fork 后的干净子进程中初始化，通过 mmap 共享缓冲区实现一次拷贝的高效 IPC。
