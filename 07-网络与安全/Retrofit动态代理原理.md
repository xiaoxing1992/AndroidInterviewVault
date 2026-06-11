---
tags: [android, network, retrofit, design-pattern, dynamic-proxy]
difficulty: S
frequency: high
---

# Retrofit 动态代理原理

## 问题

"Retrofit 是怎么把接口方法变成 HTTP 请求的？动态代理在其中扮演什么角色？ServiceMethod、CallAdapter、Converter 各自做什么？"

## 核心答案（30秒版本）

Retrofit 通过 `Proxy.newProxyInstance` 为接口创建动态代理对象。当调用接口方法时，代理拦截调用并交给 `ServiceMethod` 处理：解析方法注解生成 HTTP 请求描述，通过 `CallAdapter` 将 OkHttp 的 Call 适配为目标类型（如 RxJava Observable、Kotlin Coroutines），通过 `Converter` 完成请求体序列化和响应体反序列化。整个过程是"注解驱动 + 动态代理 + 策略模式"的经典组合。

## 深入解析

### 原理层（源码级）

**Retrofit.create() 核心实现：**

```java
// Retrofit.java
public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T) Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] { service },
        new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) {
                // 1. 如果是 Object 方法（toString/equals），直接执行
                if (method.getDeclaringClass() == Object.class) {
                    return method.invoke(this, args);
                }
                // 2. 加载或缓存 ServiceMethod
                return loadServiceMethod(method).invoke(args);
            }
        });
}
```

**ServiceMethod 解析流程：**

```java
// ServiceMethod.java — 解析注解生成请求工厂
abstract class ServiceMethod<T> {
    static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
        // 1. 解析方法注解 → RequestFactory
        RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
        // 2. 获取返回类型
        Type returnType = method.getGenericReturnType();
        // 3. 构建 HttpServiceMethod
        return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
    }
}

// RequestFactory 解析注解
class RequestFactory {
    static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
        Builder builder = new Builder(retrofit, method);
        // 解析方法级注解：@GET, @POST, @Headers 等
        for (Annotation annotation : method.getAnnotations()) {
            builder.parseMethodAnnotation(annotation);
        }
        // 解析参数级注解：@Path, @Query, @Body, @Field 等
        for (int i = 0; i < parameterCount; i++) {
            builder.parseParameter(i, parameterTypes[i], parameterAnnotations[i]);
        }
        return builder.build();
    }
}
```

**HttpServiceMethod 三种子类（Retrofit 2.9+）：**

```java
// 1. CallAdapted — 普通 CallAdapter 适配
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
    private final CallAdapter<ResponseT, ReturnT> callAdapter;
    
    @Override
    protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
        return callAdapter.adapt(call);
    }
}

// 2. SuspendForResponse — Kotlin suspend 函数返回 Response<T>
static final class SuspendForResponse<ResponseT> extends HttpServiceMethod<ResponseT, Object> {
    @Override
    protected Object adapt(Call<ResponseT> call, Object[] args) {
        // 获取 Continuation 参数
        Continuation<Response<ResponseT>> continuation = (Continuation) args[args.length - 1];
        return KotlinExtensions.awaitResponse(call, continuation);
    }
}

// 3. SuspendForBody — Kotlin suspend 函数直接返回 T
static final class SuspendForBody<ResponseT> extends HttpServiceMethod<ResponseT, Object> {
    @Override
    protected Object adapt(Call<ResponseT> call, Object[] args) {
        Continuation<ResponseT> continuation = (Continuation) args[args.length - 1];
        return KotlinExtensions.await(call, continuation);
    }
}
```

**CallAdapter 工作原理：**

```java
// CallAdapter 接口
public interface CallAdapter<R, T> {
    Type responseType();  // 响应体类型，用于选择 Converter
    T adapt(Call<R> call); // 将 Call<R> 适配为目标类型 T
}

// RxJava3CallAdapter 示例
class RxJava3CallAdapter<R> implements CallAdapter<R, Object> {
    @Override
    public Object adapt(Call<R> call) {
        Observable<Response<R>> responseObservable = new CallEnqueueObservable<>(call);
        // 根据配置决定返回 Observable<R> 还是 Observable<Response<R>>
        return isBody ? new BodyObservable<>(responseObservable) : responseObservable;
    }
}
```

**Converter 工作原理：**

```java
// Converter 接口
public interface Converter<F, T> {
    T convert(F value);
}

// GsonConverterFactory
class GsonConverterFactory extends Converter.Factory {
    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, ...) {
        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new GsonResponseBodyConverter<>(gson, adapter);
    }
    
    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type, ...) {
        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new GsonRequestBodyConverter<>(gson, adapter);
    }
}
```

**ServiceMethod 缓存机制：**

```java
// Retrofit.java
private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();

ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;
    
    synchronized (serviceMethodCache) {
        result = serviceMethodCache.get(method);
        if (result == null) {
            result = ServiceMethod.parseAnnotations(this, method);
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```

### 实战层（项目经验）

**自定义 CallAdapter（统一错误处理）：**

```kotlin
// 统一响应包装
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val code: Int, val message: String) : ApiResult<Nothing>()
}

// 自定义 CallAdapter
class ApiResultCallAdapter<R>(
    private val responseType: Type
) : CallAdapter<R, Call<ApiResult<R>>> {
    override fun responseType(): Type = responseType
    override fun adapt(call: Call<R>): Call<ApiResult<R>> = ApiResultCall(call)
}

class ApiResultCall<R>(private val delegate: Call<R>) : Call<ApiResult<R>> {
    override fun enqueue(callback: Callback<ApiResult<R>>) {
        delegate.enqueue(object : Callback<R> {
            override fun onResponse(call: Call<R>, response: Response<R>) {
                val result = if (response.isSuccessful) {
                    ApiResult.Success(response.body()!!)
                } else {
                    ApiResult.Error(response.code(), response.message())
                }
                callback.onResponse(this@ApiResultCall, Response.success(result))
            }
            override fun onFailure(call: Call<R>, t: Throwable) {
                callback.onResponse(
                    this@ApiResultCall,
                    Response.success(ApiResult.Error(-1, t.message ?: "Unknown"))
                )
            }
        })
    }
}
```

**Retrofit 与 Kotlin Coroutines 最佳实践：**

```kotlin
// 接口定义
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User
    
    @GET("users/{id}")
    suspend fun getUserResponse(@Path("id") id: String): Response<User>
}

// Repository 层
class UserRepository(private val api: ApiService) {
    suspend fun getUser(id: String): Result<User> = runCatching {
        api.getUser(id)
    }
}
```

**性能优化：Retrofit 预加载 ServiceMethod：**

```kotlin
// 应用启动时预热，避免首次调用时反射解析延迟
fun preloadServiceMethods(retrofit: Retrofit, serviceClass: Class<*>) {
    serviceClass.declaredMethods.forEach { method ->
        try {
            retrofit.loadServiceMethod(method) // 触发缓存
        } catch (e: Exception) { /* ignore default methods */ }
    }
}
```

### 延伸问题

- [[OkHttp拦截器链与连接池]] — Retrofit 底层网络请求的执行引擎
- [[OAuth2-JWT Token刷新机制]] — CallAdapter 与 Authenticator 配合实现无感刷新
- [[HTTPS证书校验与证书锁定]] — Retrofit 配置 OkHttpClient 实现安全通信

## 记忆锚点

Retrofit = 动态代理拦截方法调用 + ServiceMethod 解析注解生成请求 + CallAdapter 适配返回类型 + Converter 序列化/反序列化，一切围绕"接口即 API 文档"的设计哲学。