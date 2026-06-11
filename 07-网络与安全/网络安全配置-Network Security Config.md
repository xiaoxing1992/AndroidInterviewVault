---
tags: [android, security, network-security-config, certificate, cleartext]
difficulty: A
frequency: medium
---

# 网络安全配置 - Network Security Config

## 问题

"Android 的 Network Security Config 是什么？怎么配置 debug/release 不同的证书信任策略？如何处理自签名证书和明文流量？"

## 核心答案（30秒版本）

Network Security Config 是 Android 7.0+ 提供的声明式网络安全策略，通过 XML 配置文件控制应用的证书信任、明文流量、Certificate Pinning 等行为，无需修改代码。支持按域名配置不同策略、debug/release 差异化信任、自签名证书导入、明文流量白名单。配置文件在 `AndroidManifest.xml` 中通过 `android:networkSecurityConfig` 引用。

## 深入解析

### 原理层（源码级）

**配置加载流程：**

```java
// NetworkSecurityConfig 加载入口
// ApplicationInfo.networkSecurityConfigRes → XmlConfigSource → NetworkSecurityConfig

// ManifestConfigSource.java
class ManifestConfigSource implements ConfigSource {
    @Override
    public NetworkSecurityConfig getDefaultConfig() {
        // 解析 XML 中的 <base-config>
        return xmlSource.getDefaultConfig();
    }
    
    @Override
    public NetworkSecurityConfig getConfigForHostname(String hostname) {
        // 解析 XML 中的 <domain-config>，按域名匹配
        return xmlSource.getConfigForHostname(hostname);
    }
}

// NetworkSecurityTrustManager.java
class NetworkSecurityTrustManager implements X509TrustManager {
    @Override
    public void checkServerTrusted(X509Certificate[] certs, String authType) {
        // 根据当前连接的 hostname 获取对应配置
        NetworkSecurityConfig config = configSource.getConfigForHostname(hostname);
        // 使用配置中的 TrustAnchor 进行校验
        config.getTrustManager().checkServerTrusted(certs, authType);
    }
}
```

**Android 默认行为变化：**

| Android 版本 | 默认行为 |
|-------------|---------|
| < 7.0 | 信任用户安装的 CA 证书 |
| 7.0 - 8.1 | 仅信任系统 CA，不信任用户 CA |
| 9.0+ | 默认禁止明文 HTTP 流量 |

### 实战层（项目经验）

**完整配置示例（res/xml/network_security_config.xml）：**

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    
    <!-- 全局基础配置 -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <!-- 仅信任系统预装 CA -->
            <certificates src="system" />
        </trust-anchors>
    </base-config>
    
    <!-- 特定域名配置 -->
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
        
        <!-- Certificate Pinning -->
        <pin-set expiration="2025-06-01">
            <pin digest="SHA-256">base64EncodedPin1=</pin>
            <pin digest="SHA-256">base64EncodedPin2=</pin>
        </pin-set>
        
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </domain-config>
    
    <!-- 允许明文的域名（如本地开发服务器） -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">10.0.2.2</domain>
        <domain includeSubdomains="false">localhost</domain>
    </domain-config>
    
    <!-- Debug 专用配置 -->
    <debug-overrides>
        <trust-anchors>
            <!-- debug 构建额外信任用户安装的证书（方便抓包） -->
            <certificates src="user" />
            <!-- 信任自定义 CA（如 Charles/mitmproxy） -->
            <certificates src="@raw/debug_ca" />
        </trust-anchors>
    </debug-overrides>
    
</network-security-config>
```

**AndroidManifest.xml 引用：**

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
</application>
```

**自签名证书配置（企业内网场景）：**

```xml
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">internal.company.com</domain>
        <trust-anchors>
            <!-- 自签名 CA 证书放在 res/raw/ 目录 -->
            <certificates src="@raw/internal_ca" />
            <!-- 同时保留系统 CA 信任 -->
            <certificates src="system" />
        </trust-anchors>
    </domain-config>
</network-security-config>
```

**多环境配置策略：**

```kotlin
// build.gradle.kts
android {
    buildTypes {
        debug {
            // debug 使用宽松配置
            manifestPlaceholders["networkSecurityConfig"] = "@xml/network_security_config_debug"
        }
        release {
            // release 使用严格配置
            manifestPlaceholders["networkSecurityConfig"] = "@xml/network_security_config_release"
        }
    }
}
```

```xml
<!-- network_security_config_release.xml — 最严格 -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
    
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2025-12-31">
            <pin digest="SHA-256">primaryPin=</pin>
            <pin digest="SHA-256">backupPin=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

**常见问题排查：**

```kotlin
// 问题1：第三方 SDK 使用 HTTP 明文导致崩溃
// 解决：为特定域名开放明文
// <domain-config cleartextTrafficPermitted="true">
//     <domain>third-party-sdk.com</domain>
// </domain-config>

// 问题2：WebView 加载 HTTP 资源被阻止
// 检查方式
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    val info = NetworkSecurityPolicy.getInstance()
    val permitted = info.isCleartextTrafficPermitted("target-host.com")
    Log.d("NSC", "Cleartext permitted: $permitted")
}

// 问题3：pin-set 过期后的行为
// expiration 到期后 pinning 规则失效，回退到普通证书链校验
// 这是一个安全网，防止 pin 过期锁死用户
```

**与 OkHttp CertificatePinner 的关系：**

| 特性 | Network Security Config | OkHttp CertificatePinner |
|------|------------------------|-------------------------|
| 配置方式 | XML 声明式 | 代码编程式 |
| 作用范围 | 全应用（含 WebView） | 仅 OkHttp 请求 |
| 过期机制 | 内置 expiration 属性 | 需自行实现 |
| 动态更新 | 不支持 | 支持运行时更新 |
| 最低版本 | Android 7.0 | 无限制 |

### 延伸问题

- [[HTTPS证书校验与证书锁定]] — Certificate Pinning 的底层实现原理
- [[WebView安全-JS注入防护]] — Network Security Config 对 WebView 的影响
- [[代码混淆与反编译防护]] — 防止攻击者反编译获取 pin 值

## 记忆锚点

Network Security Config = XML 声明式网络安全策略，控制信任锚点 + 明文流量 + 证书锁定，debug-overrides 让开发调试不受影响，Android 9+ 默认禁止明文。