---
tags: [android, 面试, 架构, 组件化, S]
difficulty: S
frequency: high
---

# 组件化路由方案 - ARouter 原理

## 问题

> ARouter 是如何实现组件间无依赖跳转的？编译时做了什么？运行时路由表是如何加载的？拦截器机制是怎样的？

## 核心答案

ARouter 通过注解处理器在编译时为每个 @Route 注解的页面生成路由映射类（path→Class），运行时通过扫描 dex 或 Gradle 插件注入的方式加载所有路由表到内存 HashMap 中。跳转时根据 path 查表找到目标 Class，通过 Intent 启动。拦截器采用责任链模式，支持登录校验、降级等逻辑。

## 深入解析

### 原理层

**整体架构：**
```
编译时:
  @Route(path="/user/detail")  →  注解处理器  →  ARouter$$Group$$user.java (路由表)
  @Interceptor(priority=1)     →  注解处理器  →  ARouter$$Interceptors$$app.java

运行时:
  ARouter.init()  →  加载所有路由表到 Warehouse.routes (HashMap)
  ARouter.getInstance().build("/user/detail").navigation()
    → 查表 → 构建 Intent → 拦截器链 → startActivity
```

**编译时生成的路由表：**
```java
// 自动生成
public class ARouter$$Group$$user implements IRouteGroup {
    @Override
    public void loadInto(Map<String, RouteMeta> atlas) {
        atlas.put("/user/detail", RouteMeta.build(
            RouteType.ACTIVITY, 
            UserDetailActivity.class, 
            "/user/detail", 
            "user", 
            null,  // paramsType
            -1     // priority
        ));
    }
}

// 分组索引（按需加载）
public class ARouter$$Root$$app implements IRouteRoot {
    @Override
    public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
        routes.put("user", ARouter$$Group$$user.class);
    }
}
```

**路由表加载方式：**
```
方式1（默认）: 扫描 dex 文件，找到 com.alibaba.android.arouter.routes 包下所有类
  → ClassUtils.getFileNameByPackageName() → 反射实例化 → loadInto()
  缺点：首次启动慢（需要遍历 dex）

方式2（arouter-register 插件）: Gradle Transform 在编译时扫描所有路由类
  → 在 LogisticsCenter.init() 方法中直接插入注册代码
  → 避免运行时 dex 扫描，启动更快
```

**导航流程：**
```kotlin
ARouter.getInstance()
    .build("/user/detail")     // 1. 创建 Postcard
    .withString("userId", id)  // 2. 填充参数
    .navigation()              // 3. 开始导航
    
// 内部流程:
// → LogisticsCenter.completion(postcard)  // 查路由表，填充目标信息
// → InterceptorService.doInterceptions()  // 执行拦截器链
// → ActivityLauncher.startActivity()      // 最终跳转
```

**拦截器责任链：**
```kotlin
@Interceptor(priority = 1, name = "登录拦截器")
class LoginInterceptor : IInterceptor {
    override fun process(postcard: Postcard, callback: InterceptorCallback) {
        if (postcard.extra == NEED_LOGIN && !isLoggedIn()) {
            callback.onInterrupt(null)  // 中断导航
            // 跳转登录页
        } else {
            callback.onContinue(postcard)  // 继续
        }
    }
}
```

### 实战层

- **服务发现**：`IProvider` 接口实现跨模块服务调用，不仅限于页面跳转
- **降级策略**：`DegradeService` 处理路由未找到的情况（H5 兜底、错误页）
- **参数自动注入**：`@Autowired` + `ARouter.inject(this)` 自动从 Intent 取参数
- **多模块独立编译**：每个模块独立生成路由表，app 模块汇总
- **替代方案**：Navigation Component（官方）、WMRouter（美团）、TheRouter（货拉拉）

```kotlin
// 跨模块服务调用
@Route(path = "/pay/service")
class PayServiceImpl : PayService {
    override fun pay(orderId: String) { ... }
}

// 其他模块使用
val payService = ARouter.getInstance().navigation(PayService::class.java)
payService.pay(orderId)
```

**坑点：**
- InstantRun/热修复可能导致路由表加载失败
- 混淆需要 keep 路由表类
- 多进程场景需要注意路由表在每个进程都要初始化

### 延伸问题

- [[模块化依赖管理策略]]
- [[插件化原理-ClassLoader方案]]
- [[Navigation组件原理与深链接]]

## 记忆锚点

ARouter = 编译时建电话簿（path→Class 映射表）+ 运行时查号跳转。拦截器是跳转前的安检通道。
