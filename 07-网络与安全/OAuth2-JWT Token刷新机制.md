---
tags: [android, network, security, oauth2, jwt, token-refresh]
difficulty: S
frequency: high
---

# OAuth2 / JWT Token 刷新机制

## 问题

"OAuth2 的授权流程是怎样的？JWT 的结构和安全性如何？Token 过期后怎么无感刷新？OkHttp 中 Authenticator 和拦截器怎么配合实现？"

## 核心答案（30秒版本）

OAuth2 通过授权码模式获取 Access Token 和 Refresh Token。JWT 由 Header.Payload.Signature 三部分 Base64 编码组成，服务端通过签名验证完整性。Token 刷新策略：OkHttp Authenticator 在收到 401 时自动触发刷新，配合拦截器注入 Token。关键难点是并发请求时的刷新竞态——需要用锁或队列确保只刷新一次，其他请求等待新 Token 后重试。

## 深入解析

### 原理层（源码级）

**OAuth2 授权码模式流程：**

```
┌──────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│Client│     │Auth Server│     │Resource   │     │  User    │
│(App) │     │           │     │Server(API)│     │          │
└──┬───┘     └─────┬─────┘     └─────┬─────┘     └────┬─────┘
   │               │                  │                 │
   │ 1. 打开授权页面（redirect_uri + state）            │
   │──────────────────────────────────────────────────►│
   │               │                  │                 │
   │               │    2. 用户登录并授权                │
   │               │◄─────────────────────────────────│
   │               │                  │                 │
   │ 3. 重定向回 App（authorization_code + state）      │
   │◄──────────────│                  │                 │
   │               │                  │                 │
   │ 4. 用 code 换 token（code + client_secret）       │
   │──────────────►│                  │                 │
   │               │                  │                 │
   │ 5. 返回 access_token + refresh_token              │
   │◄──────────────│                  │                 │
   │               │                  │                 │
   │ 6. 携带 access_token 请求资源                      │
   │──────────────────────────────────►│                │
   │               │                  │                 │
   │ 7. 返回资源数据                                    │
   │◄─────────────────────────────────│                │
```

**JWT 结构解析：**

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.    ← Header (Base64URL)
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ikp.  ← Payload (Base64URL)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQ.   ← Signature

// Header
{
  "alg": "RS256",    // 签名算法
  "typ": "JWT",
  "kid": "key-id-1"  // 密钥 ID（用于密钥轮换）
}

// Payload
{
  "sub": "user123",       // 主题（用户 ID）
  "iat": 1700000000,     // 签发时间
  "exp": 1700003600,     // 过期时间（1小时后）
  "iss": "auth.example.com", // 签发者
  "aud": "api.example.com",  // 受众
  "scope": "read write"      // 权限范围
}

// Signature = RS256(Base64URL(header) + "." + Base64URL(payload), privateKey)
```

**OkHttp Authenticator 源码触发机制：**

```kotlin
// RetryAndFollowUpInterceptor.kt
override fun intercept(chain: Interceptor.Chain): Response {
    var response = realChain.proceed(request)
    
    // 循环处理重定向和认证挑战
    while (true) {
        val followUp = followUpRequest(response, exchange) ?: break
        
        // followUpRequest 内部逻辑：
        // 401 → 调用 client.authenticator.authenticate(route, response)
        // 407 → 调用 client.proxyAuthenticator.authenticate(route, response)
        
        response = realChain.proceed(followUp)
        followUpCount++
        if (followUpCount > MAX_FOLLOW_UPS) throw ProtocolException("Too many follow-up requests")
    }
    return response
}
```

### 实战层（项目经验）

**完整的 Token 刷新方案（处理并发竞态）：**

```kotlin
class TokenAuthenticator(
    private val tokenManager: TokenManager,
    private val authApi: AuthApi  // 独立的 OkHttpClient，避免循环
) : Authenticator {
    
    // 用于防止并发刷新
    private val refreshLock = Mutex()
    
    override fun authenticate(route: Route?, response: Response): Request? {
        // 防止无限重试
        if (responseCount(response) >= 3) return null
        
        val currentToken = tokenManager.getAccessToken() ?: return null
        
        // 检查是否是当前 token 导致的 401
        val requestToken = response.request.header("Authorization")
            ?.removePrefix("Bearer ")
        
        // 如果请求中的 token 已经不是当前 token，说明其他线程已刷新
        if (requestToken != currentToken) {
            // 直接用新 token 重试
            return response.request.newBuilder()
                .header("Authorization", "Bearer ${tokenManager.getAccessToken()}")
                .build()
        }
        
        // 同步刷新（只有一个线程执行刷新）
        return runBlocking {
            refreshLock.withLock {
                // 双重检查：进入锁后再次确认 token 是否已被刷新
                val latestToken = tokenManager.getAccessToken()
                if (latestToken != currentToken) {
                    // 已被其他线程刷新，直接使用新 token
                    return@runBlocking response.request.newBuilder()
                        .header("Authorization", "Bearer $latestToken")
                        .build()
                }
                
                // 执行刷新
                val refreshToken = tokenManager.getRefreshToken() ?: return@runBlocking null
                try {
                    val tokenResponse = authApi.refreshToken(
                        RefreshRequest(refreshToken)
                    )
                    tokenManager.saveTokens(tokenResponse.accessToken, tokenResponse.refreshToken)
                    
                    response.request.newBuilder()
                        .header("Authorization", "Bearer ${tokenResponse.accessToken}")
                        .build()
                } catch (e: Exception) {
                    // 刷新失败，清除 token，跳转登录
                    tokenManager.clearTokens()
                    notifySessionExpired()
                    null
                }
            }
        }
    }
    
    private fun responseCount(response: Response): Int {
        var count = 1
        var prior = response.priorResponse
        while (prior != null) {
            count++
            prior = prior.priorResponse
        }
        return count
    }
}
```

**Token 注入拦截器：**

```kotlin
class AuthInterceptor(private val tokenManager: TokenManager) : Interceptor {
    
    // 不需要 token 的接口白名单
    private val noAuthPaths = setOf("/auth/login", "/auth/register", "/auth/refresh")
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        
        // 白名单接口不注入 token
        if (noAuthPaths.any { request.url.encodedPath.contains(it) }) {
            return chain.proceed(request)
        }
        
        val token = tokenManager.getAccessToken()
        val authenticatedRequest = if (token != null) {
            request.newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
        } else {
            request
        }
        
        return chain.proceed(authenticatedRequest)
    }
}
```

**OkHttpClient 组装：**

```kotlin
// 关键：刷新 token 用的 client 不能包含 Authenticator，否则死循环
val authClient = OkHttpClient.Builder()
    .baseUrl(AUTH_BASE_URL)
    .build()

val authApi = Retrofit.Builder()
    .client(authClient)
    .baseUrl(AUTH_BASE_URL)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(AuthApi::class.java)

// 业务请求 client
val apiClient = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenManager))           // 注入 token
    .authenticator(TokenAuthenticator(tokenManager, authApi)) // 401 时刷新
    .build()
```

**Token 主动刷新策略（避免 401）：**

```kotlin
class ProactiveTokenRefresher(
    private val tokenManager: TokenManager,
    private val authApi: AuthApi
) {
    // 提前 5 分钟刷新
    private val refreshThreshold = 5 * 60 * 1000L
    
    // 作为拦截器实现主动刷新
    fun createInterceptor(): Interceptor = Interceptor { chain ->
        val token = tokenManager.getAccessToken()
        if (token != null && isTokenExpiringSoon(token)) {
            // 异步刷新，不阻塞当前请求
            refreshAsync()
        }
        chain.proceed(chain.request())
    }
    
    private fun isTokenExpiringSoon(token: String): Boolean {
        return try {
            val payload = decodeJwtPayload(token)
            val exp = payload.getLong("exp") * 1000
            val remaining = exp - System.currentTimeMillis()
            remaining in 0..refreshThreshold
        } catch (e: Exception) {
            false
        }
    }
    
    private fun decodeJwtPayload(jwt: String): JSONObject {
        val parts = jwt.split(".")
        val payload = Base64.decode(parts[1], Base64.URL_SAFE or Base64.NO_WRAP)
        return JSONObject(String(payload))
    }
}
```

**Token 安全存储：**

```kotlin
class SecureTokenManager(context: Context) : TokenManager {
    
    private val prefs = EncryptedSharedPreferences.create(
        context,
        "auth_tokens",
        MasterKey.Builder(context).setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build(),
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
    
    override fun saveTokens(accessToken: String, refreshToken: String) {
        prefs.edit()
            .putString("access_token", accessToken)
            .putString("refresh_token", refreshToken)
            .putLong("saved_at", System.currentTimeMillis())
            .apply()
    }
    
    override fun getAccessToken(): String? = prefs.getString("access_token", null)
    override fun getRefreshToken(): String? = prefs.getString("refresh_token", null)
    
    override fun clearTokens() {
        prefs.edit().clear().apply()
    }
}
```

**JWT 安全注意事项：**

| 风险 | 说明 | 应对 |
|------|------|------|
| Token 泄露 | 被中间人截获 | 强制 HTTPS + Certificate Pinning |
| Token 存储不安全 | 明文存储被读取 | EncryptedSharedPreferences |
| 过长有效期 | 泄露后长期有效 | Access Token 15-30 分钟 |
| 无法撤销 | JWT 无状态，无法主动失效 | 服务端维护黑名单 + 短有效期 |
| alg:none 攻击 | 伪造无签名 token | 服务端强制验证签名算法 |

### 延伸问题

- [[OkHttp拦截器链与连接池]] — Authenticator 在 RetryAndFollowUpInterceptor 中触发
- [[Retrofit动态代理原理]] — CallAdapter 配合 Token 刷新的错误处理
- [[数据加密方案-KeyStore与加密存储]] — Token 的安全存储方案
- [[HTTPS证书校验与证书锁定]] — 防止 Token 在传输中被截获

## 记忆锚点

OAuth2 授权码换 Token，JWT 三段式结构靠签名防篡改；刷新核心难点是并发竞态——Mutex + 双重检查确保只刷一次，其他请求等新 Token 重试；Authenticator 处理 401，拦截器注入 Token，两个 OkHttpClient 避免循环。