---
tags: [android, security, https, tls, certificate-pinning]
difficulty: S
frequency: high
---

# HTTPS 证书校验与证书锁定

## 问题

"说说 HTTPS 的 TLS 握手流程？Android 中证书链校验是怎么做的？Certificate Pinning 怎么实现？有什么风险？"

## 核心答案（30秒版本）

TLS 握手经历 ClientHello → ServerHello → 证书交换 → 密钥协商 → Finished 五步，建立加密通道。Android 默认通过系统信任的 CA 证书链逐级验证服务器证书。Certificate Pinning 是在证书链校验之上额外锁定证书公钥的 SHA-256 指纹，防止中间人攻击（即使攻击者拥有合法 CA 签发的证书）。OkHttp 通过 `CertificatePinner` 实现，需注意证书轮换时的 pin 更新策略。

## 深入解析

### 原理层（源码级）

**TLS 1.3 握手流程（简化）：**

```
Client                              Server
  |--- ClientHello ------------------>|  (支持的密码套件、随机数、key_share)
  |<-- ServerHello -------------------|  (选定密码套件、随机数、key_share)
  |<-- EncryptedExtensions -----------|
  |<-- Certificate -------------------|  (服务器证书链)
  |<-- CertificateVerify -------------|  (签名证明持有私钥)
  |<-- Finished ----------------------|
  |--- Finished --------------------->|
  |========= Application Data ========|  (对称加密通信)
```

**Android 证书链校验流程：**

```java
// TrustManagerImpl.java (Android 系统实现)
public List<X509Certificate> checkServerTrusted(X509Certificate[] chain, String authType, String host) {
    // 1. 构建证书链（补全中间证书）
    List<X509Certificate> fullChain = buildChain(chain);
    
    // 2. 验证每个证书的签名
    for (int i = 0; i < fullChain.size() - 1; i++) {
        fullChain.get(i).verify(fullChain.get(i + 1).getPublicKey());
    }
    
    // 3. 检查根证书是否在系统信任库中
    X509Certificate root = fullChain.get(fullChain.size() - 1);
    if (!trustedRoots.contains(root)) {
        throw new CertificateException("Untrusted root");
    }
    
    // 4. 检查证书有效期
    for (X509Certificate cert : fullChain) {
        cert.checkValidity();
    }
    
    // 5. 检查主机名匹配
    HostnameVerifier.verify(host, chain[0]);
    
    // 6. 检查证书吊销状态（OCSP/CRL）
    checkRevocation(fullChain);
    
    return fullChain;
}
```

**OkHttp CertificatePinner 源码：**

```kotlin
// CertificatePinner.kt
class CertificatePinner(private val pins: Set<Pin>) {
    
    fun check(hostname: String, peerCertificates: List<Certificate>) {
        val dominated = pins.filter { it.matches(hostname) }
        if (dominated.isEmpty()) return // 没有 pin 规则，跳过
        
        for (certificate in peerCertificates) {
            val x509 = certificate as X509Certificate
            // 计算证书公钥的 SHA-256
            val sha256 = sha256Hash(x509.publicKey.encoded)
            
            // 检查是否匹配任一 pin
            if (dominated.any { it.hash == sha256 }) {
                return // 匹配成功
            }
        }
        
        // 所有证书都不匹配，抛出异常
        throw SSLPeerUnverifiedException(
            "Certificate pinning failure for $hostname"
        )
    }
    
    data class Pin(
        val pattern: String,      // "*.example.com"
        val hashAlgorithm: String, // "sha256"
        val hash: ByteString       // 公钥哈希值
    )
}
```

**Pin 的计算方式：**

```kotlin
// 计算证书公钥的 SHA-256 pin
fun calculatePin(certificate: X509Certificate): String {
    val publicKeyEncoded = certificate.publicKey.encoded
    val digest = MessageDigest.getInstance("SHA-256")
    val hash = digest.digest(publicKeyEncoded)
    return "sha256/${Base64.encodeToString(hash, Base64.NO_WRAP)}"
}
```

### 实战层（项目经验）

**OkHttp 配置 Certificate Pinning：**

```kotlin
val certificatePinner = CertificatePinner.Builder()
    // 锁定叶子证书（最严格，证书轮换时必须更新）
    .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    // 锁定中间 CA（推荐，证书轮换时不需要更新）
    .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
    // 备份 pin（下一次证书轮换的公钥）
    .add("api.example.com", "sha256/CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=")
    .build()

val client = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build()
```

**动态 Pin 更新策略（避免锁死用户）：**

```kotlin
// 方案：服务端下发 pin 列表 + 本地硬编码兜底
class DynamicCertificatePinner(
    private val hardcodedPins: Set<String>,
    private val remotePinStore: PinStore
) {
    fun buildPinner(hostname: String): CertificatePinner {
        val builder = CertificatePinner.Builder()
        
        // 远程 pin（优先）
        val remotePins = remotePinStore.getPins(hostname)
        // 硬编码 pin（兜底）
        val allPins = remotePins.ifEmpty { hardcodedPins }
        
        allPins.forEach { pin ->
            builder.add(hostname, pin)
        }
        return builder.build()
    }
}
```

**自定义 TrustManager（自签名证书场景）：**

```kotlin
// 仅用于开发/测试环境，生产环境禁止
fun createUnsafeTrustManager(): X509TrustManager {
    return object : X509TrustManager {
        override fun checkClientTrusted(chain: Array<X509Certificate>, authType: String) {}
        override fun checkServerTrusted(chain: Array<X509Certificate>, authType: String) {
            // 至少验证证书有效期
            chain[0].checkValidity()
        }
        override fun getAcceptedIssuers(): Array<X509Certificate> = arrayOf()
    }
}

// 正确做法：将自签名证书加入自定义信任库
fun createCustomTrustManager(certInputStream: InputStream): X509TrustManager {
    val cf = CertificateFactory.getInstance("X.509")
    val cert = cf.generateCertificate(certInputStream)
    
    val keyStore = KeyStore.getInstance(KeyStore.getDefaultType())
    keyStore.load(null, null)
    keyStore.setCertificateEntry("custom-ca", cert)
    
    val tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm())
    tmf.init(keyStore)
    
    return tmf.trustManagers[0] as X509TrustManager
}
```

**常见安全风险与应对：**

| 风险 | 说明 | 应对 |
|------|------|------|
| Pin 过期 | 证书轮换后 pin 不匹配 | 始终包含备份 pin |
| 锁死用户 | 所有 pin 失效无法连接 | 设置 pin 最大有效期 + 远程更新 |
| 调试困难 | 抓包工具无法使用 | debug 构建禁用 pinning |
| 绕过攻击 | root 设备 hook TrustManager | 配合 SafetyNet/Play Integrity |

### 延伸问题

- [[网络安全配置-Network Security Config]] — Android 原生的证书信任配置方案
- [[OkHttp拦截器链与连接池]] — ConnectInterceptor 中 TLS 握手的触发时机
- [[数据加密方案-KeyStore与加密存储]] — 证书私钥的安全存储

## 记忆锚点

TLS 握手建立加密通道，证书链逐级验证到系统信任根，Certificate Pinning 额外锁定公钥哈希防止 CA 被攻破，务必配备备份 pin 防止锁死用户。