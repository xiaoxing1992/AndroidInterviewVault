---
tags: [android, 面试, framework, pms, A]
difficulty: A
frequency: medium
---

# PMS 包管理与安装流程

## 问题

> PackageManagerService 的职责是什么？APK 的安装流程是怎样的？PMS 是如何解析和管理应用信息的？

## 核心答案

PMS 负责 APK 的安装、卸载、信息查询、权限管理。安装流程：拷贝 APK 到 /data/app → 解析 AndroidManifest.xml 提取四大组件信息 → dex 优化（dex2oat）→ 创建数据目录 → 更新 packages.xml → 发送广播通知。PMS 启动时扫描所有已安装 APK 目录，将解析结果缓存在内存中供快速查询。

## 深入解析

### 原理层

**PMS 核心职责：**
1. **安装/卸载** — installPackage / deletePackage
2. **信息查询** — getPackageInfo / resolveIntent / queryIntentActivities
3. **权限管理** — checkPermission / grantPermission
4. **包扫描** — 启动时扫描 /system/app、/data/app 等目录

**APK 安装完整流程：**
```
1. 拷贝阶段
   APK → /data/app/{packageName}-{random}/base.apk
   解压 native so → /data/app/{packageName}-{random}/lib/

2. 解析阶段
   PackageParser.parsePackage()
   → 解析 AndroidManifest.xml
   → 提取 Activity/Service/Receiver/Provider 信息
   → 提取权限声明和请求

3. dex 优化阶段
   dex2oat 将 dex 编译为 oat（机器码）
   → /data/dalvik-cache/ 或 /data/app/.../oat/

4. 数据准备阶段
   创建 /data/data/{packageName}/ 数据目录
   设置 UID/GID 和文件权限

5. 注册阶段
   更新 /data/system/packages.xml（持久化）
   更新内存中的 PackageSetting
   注册四大组件到对应的 Resolver

6. 通知阶段
   发送 ACTION_PACKAGE_ADDED 广播
   通知 Launcher 更新图标
```

**packages.xml 结构：**
```xml
<packages>
  <package name="com.example.app" 
           codePath="/data/app/com.example.app-1"
           userId="10086"
           version="28">
    <perms>
      <item name="android.permission.INTERNET"/>
    </perms>
  </package>
</packages>
```

**PMS 启动扫描过程：**
```java
// SystemServer 启动时
PackageManagerService.main() {
    // 1. 扫描系统应用
    scanDirLI(Environment.getRootDirectory() + "/app");     // /system/app
    scanDirLI(Environment.getRootDirectory() + "/priv-app"); // /system/priv-app
    
    // 2. 扫描用户安装应用
    scanDirLI(dataDir + "/app");  // /data/app
    
    // 3. 对比 packages.xml，处理升级/卸载
    // 4. 更新权限
    updatePermissionsLocked();
}
```

**Intent 解析机制：**
- PMS 维护 `ActivityIntentResolver`、`ServiceIntentResolver` 等
- `resolveIntent()` 根据 action/category/data 匹配最佳 Activity
- 多个匹配时弹出选择器（ResolverActivity）

### 实战层

- **Split APK**：Android App Bundle 安装时拆分为 base.apk + config.apk（按语言/密度/ABI）
- **Instant App**：不安装直接运行，PMS 通过 InstantAppResolver 处理
- **包可见性（Android 11+）**：`<queries>` 声明可见的包，`queryIntentActivities` 受限
- **V2/V3 签名校验**：安装时 PMS 校验 APK 签名完整性
- **覆盖安装**：校验签名一致性 → 保留数据目录 → 替换 APK → 重新 dex2oat

### 延伸问题

- [[APK签名机制-V1-V2-V3]]
- [[热修复原理-类加载与底层替换]]
- [[多渠道打包方案]]

## 记忆锚点

PMS = 应用商店的后台管理系统。安装 = 拷贝文件 + 解析清单 + 编译优化 + 建档注册 + 通知大家。
