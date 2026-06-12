---
tags: [android, 面试, framework, system_server, service, S]
difficulty: S
frequency: high
---

# systemReady() 机制详解

## 问题

> systemReady() 是什么？每个系统服务都有吗？什么时候被调用？

## 核心答案

`systemReady()` 是系统服务的**延迟激活**机制。服务先实例化并注册到 ServiceManager（让其他服务可以引用），但不立即工作，等到所有依赖就绪后通过 `systemReady()` 统一激活。不是每个服务都有——只有有复杂依赖的服务（AMS、WMS、PMS、InputManager）需要，简单服务（BatteryService、UsageStatsService）启动后直接可用。调用发生在 `startOtherServices()` 末尾，PMS 先就绪，AMS 最关键（触发 WMS/InputManager 就绪、启动 SystemUI、启动桌面、发送开机完成广播）。

## 深入解析

### 原理层

**服务生命周期三阶段：**

```text
new XxxService()              → 阶段1：实例化（最小初始化，可被其他服务引用）
SM.addService("xxx", svc)     → 阶段2：注册到 ServiceManager（App 可以查到）
svc.systemReady()             → 阶段3：就绪（真正开始工作）
```

`systemReady()` 解决的核心问题：服务在阶段1-2就被其他服务引用了，但依赖的服务还没就绪。延迟到阶段3才真正启动业务逻辑，避免依赖缺失导致出错。

**哪些服务有 systemReady()，哪些没有：**

| 有 systemReady() | 原因 | 没有 systemReady() | 原因 |
|---|---|---|---|
| AMS | 依赖 PMS/WMS/PowerManager 全部就绪后才能调度 Activity | BatteryService | 简单监听电池状态，无复杂依赖 |
| WMS | 依赖 AMS 和 DisplayManager 就绪后才能管理窗口 | UsageStatsService | 只记录使用统计，启动后直接可用 |
| PMS | 需要等 dex 优化完成、权限数据库加载完毕 | CachedDeviceStateService | 纯状态缓存，无启动依赖 |
| InputManagerService | 依赖 WMS 就绪后才能分发输入事件 | BinderCallsStatsService | 只收集统计，无需等待 |
| AlarmManagerService | 需要恢复之前保存的 pending alarms | OverlayManagerService | 启动后直接可用 |
| ConnectivityService | 依赖 NetworkManagement 就绪 | ... | ... |

**判断标准：** 服务启动时依赖的其他服务还没就绪 → 需要 `systemReady()`。无复杂依赖、启动后直接可用 → 不需要。

### 什么时候调用

**在 `startOtherServices()` 末尾，所有服务实例化完之后集中调用。**

```text
SystemServer.run()
  → startBootstrapServices()    // 创建 AMS、PMS 等
  → startCoreServices()         // 创建 BatteryService 等
  → startOtherServices() {
      // === 前半段：创建服务，互相注入引用 ===
      WindowManagerService wm = WM.main(...);
      ams.setWindowManager(wm);
      wm.setAms(ams);
      // ...更多服务创建和引用注入

      // === 后半段：集中调用 systemReady() ===

      // 1. PMS 最先就绪
      mPackageManagerService.systemReady();

      // 2. AMS 就绪（核心，触发后续启动链）
      mActivityManagerService.systemReady(() -> {
          // 在 AMS 主线程回调中：

          // 3. 依次就绪
          mWindowManagerService.systemReady();
          mInputManager.systemReady();
          mNetworkManagementService.systemReady();
          // ...

          // 4. 启动 SystemUI
          startSystemUi();

          // 5. 启动 Home Activity（桌面）
          mAtm.startHomeOnAllDisplays(...);
      }, ...);
  }
```

**调用顺序的关键约束：**

```text
PMS.systemReady()          ← 最先（其他服务需要查包信息）
  ↓
AMS.systemReady()          ← 最关键（触发后续整个启动链）
  ├── WMS.systemReady()
  ├── InputManager.systemReady()
  ├── ConnectivityService.systemReady()
  ├── ... 其他服务按依赖顺序就绪
  ├── startSystemUi()            ← 启动状态栏、导航栏
  ├── startHomeActivity()        ← 启动桌面 Launcher
  └── send ACTION_BOOT_COMPLETED ← 标志开机完成
```

**为什么 AMS.systemReady() 最关键：**

它不只是标记 AMS 就绪，而是整个系统启动完成的触发器：
- 在回调中启动所有依赖 AMS 的其他服务
- 启动 SystemUI（状态栏、通知栏）
- 启动 Home Activity（Launcher 桌面）
- 最后发送 `ACTION_BOOT_COMPLETED` 广播
- 从此 App 进程才能正常启动和运行

### systemReady() 内部通常做什么

```java
public void systemReady() {
    // 1. 标记就绪
    mSystemReady = true;

    // 2. 处理积压请求（就绪前收到的请求先缓存，就绪后统一处理）
    for (PendingRequest r : mPendingRequests) {
        processRequest(r);
    }
    mPendingRequests.clear();

    // 3. 恢复之前保存的状态
    //    AMS：恢复 Activity 栈、前台 Service 列表
    //    PMS：恢复权限数据库、验证包完整性
    //    AlarmManager：恢复 pending alarms

    // 4. 启动依赖此服务的组件
    startDependentServices();
}
```

**积压请求处理：**

```text
时间线：
  T1: Service 注册到 SM
  T2: 其他进程通过 SM 查到 Service，发来请求
  T3: Service 还没 systemReady()，请求被缓存到 PendingRequests
  T4: systemReady() 调用
  T5: 处理所有积压请求 + 标记就绪
  T6: 后续请求正常实时处理
```

### 实战层

**面试常见追问：**

- 如果 AMS.systemReady() 之前有人调用 AMS 的方法会怎样？
- systemReady() 和 onCreate() 的区别是什么？
- 为什么 PMS 要比 AMS 先 systemReady？
- systemReady() 是同步的还是异步的？
- 如果一个服务的 systemReady() 很慢，会影响开机速度吗？
- ACTION_BOOT_COMPLETED 是在哪个 systemReady() 之后发送的？

**systemReady() 前调用服务会怎样？**

```text
AMS 还没 systemReady() 时，App 通过 Binder 调用 AMS.startActivity()
  → Binder 线程收到请求
  → AMS 检查 mSystemReady == false
  → 请求被缓存到 mPendingRequests
  → 等 AMS.systemReady() 后统一处理
  // 或者：
  → 直接抛异常 / 返回错误码
```

**systemReady() 和 onCreate() 的区别：**

| | 构造函数 / onCreate() | systemReady() |
|---|---|---|
| 时机 | 服务实例化时 | 所有服务就绪后 |
| 做什么 | 最小初始化、注册 Binder | 激活业务、恢复状态、启动依赖组件 |
| 依赖其他服务 | 尽量不依赖（其他服务可能还没创建） | 可以安全调用其他服务 |
| 类比 | "搬进办公室" | "开始办公" |

**排查和观察：**

```bash
# 查看开机完成时间（ACTION_BOOT_COMPLETED 的时间戳）
adb logcat -b events | grep boot_progress_enable_screen

# 查看 systemReady 耗时
adb logcat -s SystemServer | grep -i "systemReady\|ready"

# 查看开机各阶段耗时
adb logcat -b events | grep boot_progress

# 查看是否收到开机完成广播
adb logcat | grep BOOT_COMPLETED
```

**容易答错的点：**

- 不是每个系统服务都有 `systemReady()`，只有有复杂依赖的服务需要
- `systemReady()` 不是构造函数的一部分，它是独立的延迟激活调用
- AMS.systemReady() 的回调是整个系统启动完成的标志，不只是 AMS 就绪
- `ACTION_BOOT_COMPLETED` 是在 AMS.systemReady() 回调的最后发出的
- 就绪前收到的请求会被缓存，不是丢弃

### 延伸问题

- [[系统服务启动流程]]
- [[系统服务依赖解决与发布机制与线程模型]]
- [[SystemServer启动流程]]
- [[App冷启动全链路]]
- [[Activity启动流程]]

## 记忆锚点

`systemReady()` = **延迟激活**。先创建注册（让其他服务能引用），依赖都就绪后再调 `systemReady()` 开始工作。不是每个服务都有，只有有复杂依赖的才有。AMS.systemReady() 最关键——它触发 SystemUI、桌面、开机完成广播。
