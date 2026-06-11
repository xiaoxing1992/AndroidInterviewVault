---
tags: [android, security, webview, javascript, xss, injection]
difficulty: A
frequency: medium
---

# WebView 安全 - JS 注入防护

## 问题

"addJavascriptInterface 有什么安全漏洞？怎么设计安全的 JS Bridge？WebView 的同源策略和沙箱机制是怎样的？"

## 核心答案（30秒版本）

Android 4.2 以下 `addJavascriptInterface` 存在远程代码执行漏洞：攻击者可通过反射调用 Java 任意方法。4.2+ 通过 `@JavascriptInterface` 注解限制暴露方法解决了此问题。安全的 JS Bridge 方案包括：URL Scheme 拦截（shouldOverrideUrlLoading）、prompt/console 拦截、postMessage 通信。WebView 遵循同源策略但默认允许 file:// 跨域访问，需手动关闭。沙箱机制通过独立进程 + 渲染器隔离降低漏洞影响。

## 深入解析

### 原理层（源码级）

**addJavascriptInterface 漏洞原理（Android < 4.2）：**

```java
// 漏洞代码
webView.addJavascriptInterface(new Object() {
    public void doSomething() { /* ... */ }
}, "bridge");

// 攻击者注入的 JS（利用 Java 反射）
// 通过 getClass() 获取 Class 对象，进而调用任意方法
var exploit = function() {
    // 获取 Runtime 对象
    var runtime = bridge.getClass().forName("java.lang.Runtime")
        .getMethod("getRuntime", null).invoke(null, null);
    // 执行任意命令
    runtime.exec("rm -rf /data/data/com.victim.app/");
};
```

**4.2+ 修复方案源码：**

```java
// WebViewClassic.java (Android 4.2+)
private boolean canInvokeMethod(Method method) {
    // 只允许调用带 @JavascriptInterface 注解的方法
    return method.isAnnotationPresent(JavascriptInterface.class);
}

// 安全的接口定义
class SafeBridge {
    @JavascriptInterface  // 必须有此注解
    public String getData() {
        return "safe data";
    }
    
    // 没有注解的方法，JS 无法调用
    public void internalMethod() { /* ... */ }
}
```

**WebView 同源策略相关设置：**

```java
// WebSettings 安全相关源码
public abstract class WebSettings {
    // file:// URL 是否允许访问其他 file:// 资源
    public abstract void setAllowFileAccessFromFileURLs(boolean flag);
    // 默认值：API 15 以下 true，API 16+ false
    
    // file:// URL 是否允许访问任何来源的内容
    public abstract void setAllowUniversalAccessFromFileURLs(boolean flag);
    // 默认值：API 15 以下 true，API 16+ false
    
    // 是否允许访问 file:// 协议
    public abstract void setAllowFileAccess(boolean flag);
    // 默认值：API 29 以下 true，API 30+ false
    
    // 是否允许访问 content:// 协议
    public abstract void setAllowContentAccess(boolean flag);
}
```

**WebView 多进程架构（Android 8.0+）：**

```
┌─────────────────────┐     ┌─────────────────────┐
│   App Process        │     │  Renderer Process    │
│                      │     │  (沙箱化)            │
│  WebView API         │◄───►│  Blink 渲染引擎      │
│  WebViewClient       │ IPC │  V8 JS 引擎          │
│  WebChromeClient     │     │  网络请求             │
│                      │     │                      │
│  权限：完整应用权限   │     │  权限：最小化         │
└─────────────────────┘     └─────────────────────┘
```

### 实战层（项目经验）

**安全的 JS Bridge 方案一：URL Scheme 拦截**

```kotlin
class SecureWebViewClient : WebViewClient() {
    
    private val handlers = mutableMapOf<String, (Map<String, String>) -> String?>()
    
    init {
        handlers["getUserInfo"] = { params -> handleGetUserInfo(params) }
        handlers["shareContent"] = { params -> handleShare(params) }
    }
    
    override fun shouldOverrideUrlLoading(view: WebView, request: WebResourceRequest): Boolean {
        val url = request.url
        if (url.scheme == "jsbridge") {
            val method = url.host ?: return true
            val params = url.queryParameters()
            val callbackId = url.getQueryParameter("_callback_id")
            
            // 白名单校验
            val handler = handlers[method] ?: run {
                Log.w("Bridge", "Unknown method: $method")
                return true
            }
            
            // 执行并回调
            val result = handler(params)
            callbackId?.let {
                view.evaluateJavascript(
                    "window.__bridge_callbacks['$it']($result)", null
                )
            }
            return true
        }
        return super.shouldOverrideUrlLoading(view, request)
    }
}

// JS 端调用
// window.callNative = function(method, params, callback) {
//     var callbackId = 'cb_' + Date.now();
//     window.__bridge_callbacks[callbackId] = callback;
//     var url = 'jsbridge://' + method + '?_callback_id=' + callbackId;
//     Object.keys(params).forEach(k => url += '&' + k + '=' + encodeURIComponent(params[k]));
//     var iframe = document.createElement('iframe');
//     iframe.src = url;
//     document.body.appendChild(iframe);
//     setTimeout(() => iframe.remove(), 0);
// };
```

**安全的 JS Bridge 方案二：evaluateJavascript 双向通信**

```kotlin
class ModernJsBridge(private val webView: WebView) {
    
    private val pendingCallbacks = ConcurrentHashMap<String, (String) -> Unit>()
    
    // Native 调用 JS
    fun callJs(method: String, params: JSONObject, callback: (String) -> Unit) {
        val callId = UUID.randomUUID().toString()
        pendingCallbacks[callId] = callback
        
        val script = "window.__nativeBridge.onNativeCall('$callId', '$method', ${params})"
        webView.evaluateJavascript(script) { result ->
            // evaluateJavascript 的返回值是同步结果
        }
    }
    
    // JS 调用 Native（通过 @JavascriptInterface）
    @JavascriptInterface
    fun postMessage(messageJson: String) {
        val message = JSONObject(messageJson)
        val type = message.getString("type")
        
        when (type) {
            "call" -> handleJsCall(message)
            "response" -> handleJsResponse(message)
        }
    }
    
    private fun handleJsCall(message: JSONObject) {
        val method = message.getString("method")
        val callId = message.getString("callId")
        val params = message.getJSONObject("params")
        
        // 权限校验
        if (!isMethodAllowed(method, webView.url)) {
            respondToJs(callId, error = "Permission denied")
            return
        }
        
        // 分发处理
        val result = dispatchMethod(method, params)
        respondToJs(callId, result = result)
    }
    
    // 域名白名单校验
    private fun isMethodAllowed(method: String, url: String?): Boolean {
        val host = Uri.parse(url).host ?: return false
        val allowedHosts = setOf("www.example.com", "m.example.com")
        return host in allowedHosts
    }
}
```

**WebView 安全配置清单：**

```kotlin
fun configureSecureWebView(webView: WebView) {
    val settings = webView.settings
    
    // 1. 禁用危险的文件访问
    settings.allowFileAccess = false
    settings.allowFileAccessFromFileURLs = false
    settings.allowUniversalAccessFromFileURLs = false
    settings.allowContentAccess = false
    
    // 2. 禁用不需要的功能
    settings.setGeolocationEnabled(false)
    settings.savePassword = false
    settings.saveFormData = false
    
    // 3. 启用安全特性
    settings.mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW // 禁止混合内容
    settings.safeBrowsingEnabled = true  // 启用安全浏览
    
    // 4. 限制 JavaScript（如果不需要）
    settings.javaScriptEnabled = true // 按需开启
    settings.javaScriptCanOpenWindowsAutomatically = false
    
    // 5. 设置安全的 WebViewClient
    webView.webViewClient = object : WebViewClient() {
        override fun shouldOverrideUrlLoading(view: WebView, request: WebResourceRequest): Boolean {
            val url = request.url
            // 白名单域名校验
            val allowedHosts = setOf("www.example.com", "cdn.example.com")
            if (url.host !in allowedHosts) {
                // 非白名单域名，用外部浏览器打开
                context.startActivity(Intent(Intent.ACTION_VIEW, url))
                return true
            }
            return false
        }
        
        override fun onReceivedSslError(view: WebView, handler: SslErrorHandler, error: SslError) {
            // 绝对不能调用 handler.proceed()！
            handler.cancel()
            // 展示错误页面
            view.loadData(SSL_ERROR_HTML, "text/html", "utf-8")
        }
    }
}
```

**XSS 防护：输入过滤与 CSP**

```kotlin
// 服务端返回 CSP 头
// Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'

// 客户端注入内容时转义
fun sanitizeForHtml(input: String): String {
    return input
        .replace("&", "&amp;")
        .replace("<", "&lt;")
        .replace(">", "&gt;")
        .replace("\"", "&quot;")
        .replace("'", "&#x27;")
}

// 使用 evaluateJavascript 时防注入
fun safeEvaluate(webView: WebView, data: String) {
    // 使用 JSON 序列化而非字符串拼接
    val jsonData = JSONObject().put("content", data).toString()
    webView.evaluateJavascript("window.receiveData($jsonData)", null)
}
```

**Intent Scheme 攻击防护：**

```kotlin
override fun shouldOverrideUrlLoading(view: WebView, request: WebResourceRequest): Boolean {
    val url = request.url.toString()
    
    // 防止 intent:// scheme 攻击
    if (url.startsWith("intent://")) {
        try {
            val intent = Intent.parseUri(url, Intent.URI_INTENT_SCHEME)
            // 必须添加安全限制
            intent.addCategory(Intent.CATEGORY_BROWSABLE)
            intent.component = null  // 禁止指定组件
            intent.selector = null   // 禁止 selector
            
            // 校验目标应用是否存在
            if (intent.resolveActivity(context.packageManager) != null) {
                context.startActivity(intent)
            }
        } catch (e: Exception) {
            Log.e("WebView", "Invalid intent URL: $url")
        }
        return true
    }
    return false
}
```

### 延伸问题

- [[网络安全配置-Network Security Config]] — WebView 的网络安全策略受 NSC 控制
- [[代码混淆与反编译防护]] — @JavascriptInterface 方法需要 keep 规则
- [[HTTPS证书校验与证书锁定]] — WebView 的 SSL 错误处理策略

## 记忆锚点

WebView 安全三要素：@JavascriptInterface 限制暴露面 + 域名白名单控制通信范围 + 关闭 file:// 跨域访问；JS Bridge 优选 URL Scheme 拦截或 postMessage，避免直接暴露 Java 对象。