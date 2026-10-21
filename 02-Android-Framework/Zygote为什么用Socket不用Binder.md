---
tags: [android, 面试, framework, zygote, binder, ipc, S]
difficulty: S
frequency: high
---

# Zygote 为什么用 Socket 而不用 Binder

## 问题

> Zygote 和 system_server（AMS）之间的 IPC 为什么用 Unix domain socket 而不是 Binder？如果用 Binder 会有什么问题？

## 核心答案

**Binder 服务端必须是多线程的，而 Zygote 必须在 `fork()` 的那一刻保持单线程** —— 这是根本冲突，也是唯一的一级原因。Binder 接收事务靠 Binder 线程池（默认上限 15 个），线程池一启动，Zygote 就成了多线程进程；而 `fork()` 只复制调用线程，其他线程持有的锁在子进程里会永远锁死。

第二个原因是 **`binder_proc` 是每进程一份**：它在 `binder_open()` 时创建，Zygote 若持有 `/dev/binder` 的连接，fork 出的子进程继承的 fd 会指向**父进程的 `binder_proc`**，身份校验、引用计数、todo 队列全部错位。所以 Zygote 全程不碰 `/dev/binder`，每个子进程第一次调 `ProcessState::self()` 时才各自 `open`。

第三，Zygote 的通信模型极简（一次请求 → 一次 fork → 回一个 pid），`select` 单线程循环完全够用，Binder 的 RPC、对象引用、死亡通知一样都用不上。

> ⚠️ **辟谣**：网上常见的「Zygote 启动时 ServiceManager 还没就绪」是**错的**。`servicemanager` 属于 `class core`，`zygote` 属于 `class main`，前者先启动。详见下文。

## 深入解析

### 原理层

**原因一：Binder 服务端必须多线程，与 fork 的单线程前提冲突**

这是核心。Zygote 的安全约束不是"最好单线程"，而是"fork 的那一刻必须是单线程"：

```text
Zygote 使用 Binder
  → 必须 startThreadPool() 启动 Binder 线程池（至少 1 个额外线程）
  → 进程变为多线程
  → fork() 只复制调用线程，其他线程在子进程中不存在
  → 但那些线程持有的锁被完整复制过来了
  → 子进程里锁是 locked 状态，持有者却不存在
  → 子进程第一次 malloc() / 碰 ART 堆 → 死锁
```

ART 自己也起 4 个 daemon（FinalizerDaemon、ReferenceQueueDaemon、HeapTaskDaemon、Signal Catcher），但它们靠 `ZygoteHooks.preFork()` 在 fork 前被 `Daemons.stop()` 干净停掉，fork 完 `postFork()` 再拉起 —— 这叫**制造瞬时单线程**。

Binder 线程池做不到这一点：15 个线程阻塞在 `ioctl` 里等事务，你没法在 fork 前把它们"干净地停住"，强行停会丢事务、留半锁状态。**这就是 Zygote 绝不能用 Binder 当服务端的根本原因。**

**原因二：`binder_proc` 是每进程一份，fork 会串味**

Binder 驱动为每个打开 `/dev/binder` 的进程维护一个 `binder_proc`：

```text
内核 Binder 驱动状态：
  ├── binder_proc     ← 每进程一份，open("/dev/binder") 时创建
  ├── binder_thread   ← 每线程一份，按 tid 索引
  ├── binder_node     ← 服务端实体
  ├── binder_ref      ← 客户端引用（带引用计数）
  ├── todo 队列        ← 待分发事务
  └── 死亡通知链表
```

如果 Zygote 自己开了 Binder：

- fork 后子进程继承 fd，而 fd 指向同一个 `struct file`，也就是**父进程的 `binder_proc`**
- 驱动会把子进程归入父进程的身份上下文，`UID/PID` 校验、引用计数、todo 队列全部错位
- 子进程的线程在驱动侧没有对应 `binder_thread`，会在父进程的 `binder_proc` 下被"补建"出来

注意：这**不会损坏驱动本身**，但会让语义彻底错乱 —— 子进程被当成父进程，权限校验失去意义。

Android 的解法是**延迟初始化**：Zygote 从不 `open("/dev/binder")`，`ProcessState` 单例在子进程第一次调用时才创建，各自拿到独立的 `binder_proc`。

```text
Zygote（无 Binder，单线程）
  → fork()
    → 子进程返回
      → handleChildProc()
        → RuntimeInit.zygoteInit()
          → nativeZygoteInit()   // ★ 此时才 ProcessState::self()->startThreadPool()
          → ActivityThread.main()
            → attach()
              → AMS.attachApplication()  // ★ 子进程此时才通过 Binder 和 AMS 通信
```

所以准确说法是「**延迟初始化**」，不是「fork 后重新初始化」—— Zygote 压根没初始化过。

**原因三：socket 对 Zygote 完全够用**

```text
通信内容：
  AMS → Zygote：一串文本参数（uid/gid/进程名/运行时标志等）
  Zygote → AMS：一个 pid + 成功/失败状态

通信模式：
  一对一、串行、请求-应答
  不需要并发、不需要回调、不需要对象引用
```

Binder 的能力（RPC 方法调用、对象引用、死亡通知、多客户端并发）对 Zygote 来说一样用不上。

**原因四：select 循环天然匹配 Zygote 的工作模式**

```text
ZygoteServer.runSelectLoop()
  → epoll/select 监听 zygote socket
  → 有新连接/新数据
    → 串行处理：读参数 → fork → 回复
    → 回到 select 继续等待
```

串行处理保证了：
- 不会在 fork 过程中被另一个请求打断
- 每次 fork 时进程状态是确定的
- 不需要加锁，因为只有主线程在跑

### 两个流传很广的错误说法

**❌ 错误说法一：「Zygote 启动时 ServiceManager 尚未就绪」**

事实正好相反 —— **ServiceManager 比 Zygote 启动得更早**。

`init` 按 class 顺序启动服务，顺序是 `early_hal → hal → core → main → late_start`：

```rc
# servicemanager.rc（AOSP）
service servicemanager /system/bin/servicemanager
    class core animation      ← core，在 on boot 里 class_start core
    user system
    group system readproc
    critical
    onrestart class_restart --only-enabled main   ← 依赖方向：SM 重启会拉起 main（含 zygote）

# init.zygote64.rc（AOSP）
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main                ← main，在 on nonencrypted 里 class_start main
    socket zygote stream 660 root system
```

```rc
# init.rc（AOSP）
on boot
    class_start hal
    class_start core          ← servicemanager 在这里启动

on nonencrypted
    class_start main          ← zygote 在这里启动
    class_start late_start
```

三条独立证据：

1. **class 分组**：SM 在 `core`，zygote 在 `main`，`core` 先启动
2. **依赖方向**：`servicemanager.rc` 里 `onrestart class_restart --only-enabled main` —— SM 重启会把整个 main class（含 zygote）拉起来，说明 zygote 依赖 SM，不是反过来
3. **实测 PID**：`adb shell ps -A -o PID,NAME | grep -E "servicemanager|zygote"`，SM 的 PID 明显更小

而且 SM 被标记为 `critical`，崩溃会触发设备重启 —— 这种基石服务不可能晚于 zygote。

**❌ 错误说法二：「会形成循环依赖（Zygote → SM → AMS → Zygote）」**

这个链根本不构成环：

- Zygote 向 SM 注册是**一次性的客户端操作**，注册完就不再依赖 SM
- AMS 调 Zygote 是**单向**的，Zygote 从不回调 AMS
- 所以依赖图是 `SM ← Zygote`，`AMS → Zygote`，没有闭环

真正的麻烦是**自举时序**：Zygote 要在 fork 前拉起线程池才能接住 AMS 的调用，但拉起线程池就破坏了单线程前提。这是「原因一」的推论，不是独立原因。

**如果强行用 Binder 会怎样？**

| 问题 | 后果 |
|------|------|
| Binder 线程池让 Zygote 变多线程 | **fork 不安全，子进程死锁（致命）** |
| 子进程继承指向父进程 `binder_proc` 的 fd | **身份校验失效，事务归属错乱（致命）** |
| Zygote 必须在 fork 前拉起线程池才能接 AMS 调用 | 自举时序矛盾（原因一的推论） |
| 引入 Binder 依赖 | 无收益，select 已完全够用 |

### 实战层

**面试常见追问：**

- socket 和 Binder 的本质区别是什么？
- App 进程和 AMS 用的是什么通信方式？为什么 App 可以用 Binder？
- Zygote 的 socket 是谁创建的？init 进程在其中扮演什么角色？
- fork 之后子进程的文件描述符是怎么处理的？
- 为什么 `nativeZygoteInit()` 必须在子进程中调用而不是 Zygote 中？

**App 进程和 AMS 通信方式对比：**

| 通信链路 | 方式 | 原因 |
|----------|------|------|
| AMS ↔ Zygote | Unix domain socket | Zygote 需要单线程 fork 安全 |
| AMS ↔ App 进程 | Binder | App 是 fork 之后才初始化 Binder 线程池，多线程安全 |
| AMS ↔ system_server 内部服务 | 直接方法调用 | 同进程 |

**排查和观察：**

```bash
# 验证启动顺序：servicemanager 的 PID 一定小于 zygote
adb shell ps -A -o PID,NAME | grep -E "servicemanager|zygote"

# 查看 zygote socket（init 创建并传给 Zygote 的 fd）
adb shell ls -la /dev/socket/zygote

# 查看 zygote 的 rc 配置
adb shell cat /system/etc/init/hw/init.zygote64.rc
# 关键行：class main / socket zygote stream 660 root system

# 看 App 进程的 Binder fd（fork 后才存在）
adb shell "ls -la /proc/$(pidof com.example.app)/fd | grep binder"
```

**源码入口：**

- socket 注册：`ZygoteServer.registerServerSocketFromEnv()`
- select 循环：`ZygoteServer.runSelectLoop()`
- Binder 初始化：`RuntimeInit.java` → `zygoteInit()` → `ZygoteInit.nativeZygoteInit()`
- fork 前置处理：`ZygoteHooks.preFork()` / `postFork()`
- rc 定义：`system/core/rootdir/init.zygote64.rc`、`frameworks/native/cmds/servicemanager/servicemanager.rc`

**容易答错的点：**

- ❌ 不是"Binder 太重了" —— 问题不是性能，而是多线程 fork 安全性
- ❌ 不是"socket 更快" —— 两者性能差距对 Zygote 的低频通信场景不重要
- ❌ 不是"ServiceManager 还没就绪" —— SM 比 Zygote 早启动，这条是错的
- ❌ 不是"循环依赖" —— 依赖图没有闭环，真正的问题是线程模型
- ✅ App 进程可以用 Binder，因为它是 fork 之后才初始化 Binder 线程池
- ✅ 说「延迟初始化」比说「重新初始化」准确 —— Zygote 从没初始化过 Binder

### 延伸问题

- [[为什么App由Zygote-fork而非system_server]]
- [[Zygote进程启动与App进程孵化]]
- [[ZygoteConnection-runOnce详解]]
- [[Binder通信原理]]
- [[Android进程创建方式-fork-execve与Zygote]]

## 记忆锚点

Zygote 不用 Binder，就两条硬理由：**多线程不能 fork**（Binder 服务端必须有线程池，fork 只复制调用线程，锁会死锁）、**`binder_proc` 每进程一份**（fork 出的子进程会指向父进程的 `binder_proc`，身份错乱）。

socket 是普通 fd，能安全 fork、能安全关闭、不需要注册、不需要线程池 —— 对一个「只做 fork 一件事」的进程，这就是最优解。

一句话：**Binder 是给「长期提供多种服务」的进程用的，Zygote 只做「复制自己」这一件事，用不上也不需要。**
