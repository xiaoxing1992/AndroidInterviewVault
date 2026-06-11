---
tags: [android, network, okhttp, design-pattern, connection-pool]
difficulty: S
frequency: high
---

# OkHttp 拦截器链与连接池

## 问题

"说说 OkHttp 的拦截器链是怎么工作的？五大内置拦截器各自负责什么？连接池复用机制是怎样的？"

## 核心答案（30秒版本）

OkHttp 采用责任链模式，通过 `RealInterceptorChain` 将请求依次传递给每个拦截器处理。五大内置拦截器按顺序为：RetryAndFollowUpInterceptor（重试与重定向）、BridgeInterceptor（补全请求头）、CacheInterceptor（缓存策略）、ConnectInterceptor（建立连接）、CallServerInterceptor（发送请求读取响应）。ConnectionPool 通过维护空闲连接队列，对相同 Address 的连接进行复用，默认保持 5 个空闲连接、存活 5 分钟，由后台清理线程定期回收。

## 深入解析

### 原理层（源码级）

**责任链模式实现：**

```kotlin
// RealInterceptorChain.kt 核心逻辑
class RealInterceptorChain(
    private val interceptors: List<Interceptor>,
    private val index: Int,
    // ...
) : Interceptor.Chain {

    override fun proceed(request: Request): Response {
        // 构造下一个 Chain，index + 1
        val next = copy(index = index + 1)
        val interceptor = interceptors[index]
        // 当前拦截器处理，内部调用 next.proceed() 传递给下一个
        val response = interceptor.intercept(next)
        return response
    }
}
```

**五大内置拦截器职责：**

1. **RetryAndFollowUpInterceptor**：处理连接失败重试、HTTP 重定向（301/302/307/308）、认证挑战（401/407），最多重定向 20 次。

2. **BridgeInterceptor**：补全 Content-Type、Content-Length、Host、Connection、Accept-Encoding（gzip）、Cookie 等请求头；响应时处理 gzip 解压。

3. **CacheInterceptor**：基于 HTTP 缓存语义（Cache-Control、ETag、Last-Modified）判断是否使用缓存，实现强缓存与协商缓存。

4. **ConnectInterceptor**：从 ConnectionPool 获取可复用连接或创建新连接，完成 TCP + TLS 握手，建立 HTTP/2 或 HTTP/1.1 流。

5. **CallServerInterceptor**：真正发送请求数据、读取响应数据的地方，处理 HTTP 协议细节。

**ConnectionPool 源码分析：**

```kotlin
// ConnectionPool 默认配置
class ConnectionPool(
    maxIdleConnections: Int = 5,      // 最大空闲连接数
    keepAliveDuration: Long = 5,      // 存活时间
    timeUnit: TimeUnit = TimeUnit.MINUTES
)

// RealConnectionPool 清理逻辑
class RealConnectionPool {
    private val connections = ConcurrentLinkedQueue<RealConnection>()
    
    // 清理线程
    private val cleanupRunnable = Runnable {
        while (true) {
            val waitNanos = cleanup(System.nanoTime())
            if (waitNanos == -1L) return@Runnable
            // 等待指定时间后再次清理
            wait(waitNanos)
        }
    }
    
    fun cleanup(now: Long): Long {
        // 遍历连接，统计空闲连接数和空闲时间最长的连接
        // 如果空闲连接数 > maxIdleConnections 或空闲时间 > keepAliveDuration
        // 则移除该连接
    }
}
```

**连接复用判断条件（RealConnection.isEligible）：**

```kotlin
fun isEligible(address: Address, routes: List<Route>?): Boolean {
    // 1. 连接未达到最大并发流数（HTTP/2 默认 256）
    // 2. address 匹配（host、port、proxy、protocols、sslSocketFactory 等）
    // 3. HTTP/2 可合并连接（相同 IP + 证书覆盖目标域名）
    return allocations.size < allocationLimit &&
           address.equalsNonHost(this.route.address) &&
           address.url.host == this.route.address.url.host
}
```

### 实战层（项目经验）

**自定义拦截器实践：**

```kotlin
// 日志拦截器
class LoggingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val startTime = System.nanoTime()
        
        Log.d("HTTP", "Sending: ${request.method} ${request.url}")
        
        val response = chain.proceed(request)
        val duration = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)
        
        Log.d("HTTP", "Received: ${response.code} (${duration}ms)")
        return response
    }
}

// Token 注入拦截器（Application Interceptor）
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer ${tokenProvider.getToken()}")
            .build()
        return chain.proceed(request)
    }
}

// 使用方式
val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenProvider))       // Application 拦截器
    .addNetworkInterceptor(LoggingInterceptor())          // Network 拦截器
    .connectionPool(ConnectionPool(10, 3, TimeUnit.MINUTES))
    .build()
```

**Application Interceptor vs Network Interceptor 区别：**

| 特性 | Application Interceptor | Network Interceptor |
|------|------------------------|-------------------|
| 位置 | 最外层，第一个执行 | ConnectInterceptor 之后 |
| 重定向 | 只调用一次 | 每次网络请求都调用 |
| 缓存响应 | 能观察到 | 看不到缓存命中 |
| 短路能力 | 可以不调用 proceed | 必须调用 proceed |

**连接池调优经验：**

- 高并发场景（IM、直播）：增大 maxIdleConnections 到 15-20
- 省电场景：缩短 keepAliveDuration 到 1-2 分钟
- HTTP/2 场景：单连接多路复用，连接池意义降低但仍需保留

### 延伸问题

- [[Retrofit动态代理原理]] — Retrofit 如何基于 OkHttp 构建上层抽象
- [[HTTPS证书校验与证书锁定]] — ConnectInterceptor 中 TLS 握手细节
- [[OAuth2-JWT Token刷新机制]] — 自定义 Authenticator 与拦截器配合

## 记忆锚点

责任链五兄弟：重试→桥接→缓存→连接→发送，ConnectionPool 以 Address 为 key 复用空闲连接，默认 5 连接 5 分钟。
