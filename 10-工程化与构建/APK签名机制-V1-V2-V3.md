---
tags: [android, 面试, 工程化, 签名, A]
difficulty: A
frequency: medium
---

# APK 签名机制 - V1/V2/V3

## 问题

> Android APK 的 V1、V2、V3 签名方案分别是什么原理？为什么需要多种签名方案？V2 签名如何保证 APK 完整性？

## 核心答案

V1（JAR 签名）对每个文件单独签名，校验慢且不覆盖 ZIP 元数据；V2（APK Signature Scheme）对整个 APK 文件做完整性校验（分块 hash），安装更快且覆盖全部内容；V3 在 V2 基础上支持密钥轮换（Key Rotation）。Android 7.0+ 优先校验 V2，V1 作为向后兼容。三者可以同时存在。

## 深入解析

### 原理层

**V1 签名（JAR Signing）：**
```
META-INF/
├── MANIFEST.MF    — 每个文件的 SHA-256 摘要
├── CERT.SF        — MANIFEST.MF 各条目的摘要
└── CERT.RSA       — CERT.SF 的签名 + 证书

校验过程:
1. 验证 CERT.RSA 中的签名（用证书公钥）
2. 验证 CERT.SF 中的摘要与 MANIFEST.MF 一致
3. 验证 MANIFEST.MF 中的摘要与实际文件一致

问题:
- 不覆盖 META-INF 目录本身（可以往里加文件）
- 不覆盖 ZIP 元数据（Central Directory）
- 需要解压每个文件计算 hash，安装慢
```

**V2 签名（APK Signature Scheme v2）：**
```
APK 文件结构:
┌─────────────────────────┐
│ ZIP 内容 (Section 1)    │ ← 被签名
├─────────────────────────┤
│ APK Signing Block       │ ← 签名数据存放处（不被签名）
│ (Section 2)             │
├─────────────────────────┤
│ Central Directory       │ ← 被签名
│ (Section 3)             │
├─────────────────────────┤
│ End of Central Dir      │ ← 被签名
│ (Section 4)             │
└─────────────────────────┘

签名过程:
1. 将 APK 分为 Section 1/3/4（跳过 Signing Block）
2. 每个 Section 分成 1MB 的 chunk
3. 计算每个 chunk 的摘要
4. 对所有摘要的摘要进行签名
5. 签名结果存入 APK Signing Block

校验: 不需要解压 APK，直接对文件分块校验
     → 安装速度显著提升
```

**V3 签名（密钥轮换）：**
```
V3 在 V2 基础上增加:
- Proof-of-rotation 结构
- 记录签名密钥的历史链（旧密钥 → 新密钥）
- 允许 App 更换签名密钥而不影响升级

密钥轮换链:
Key_A (原始) → Key_B (第一次轮换) → Key_C (当前)
每次轮换由旧密钥签名新密钥，形成信任链
```

**V4 签名（Android 11+）：**
```
- 基于 fs-verity 的增量安装签名
- 支持 ADB 增量安装（streaming install）
- 不替代 V2/V3，是补充方案
```

### 实战层

**签名配置：**
```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file("release.keystore")
            storePassword = System.getenv("STORE_PASSWORD")
            keyAlias = "release"
            keyPassword = System.getenv("KEY_PASSWORD")
            // V1 和 V2 默认都启用
            enableV1Signing = true
            enableV2Signing = true
            enableV3Signing = true
            enableV4Signing = true
        }
    }
}
```

**验证签名：**
```bash
# 查看 APK 使用了哪些签名方案
apksigner verify -v app-release.apk

# 输出示例:
# Verified using v1 scheme (JAR signing): true
# Verified using v2 scheme (APK Signature Scheme v2): true
# Verified using v3 scheme (APK Signature Scheme v3): true
```

**签名相关的安全问题：**
- 密钥泄露 → 攻击者可以签名恶意 APK 伪装成你的 App
- Google Play App Signing → 将签名密钥托管给 Google，降低泄露风险
- 密钥丢失 → V3 之前无法恢复，App 必须重新上架

**多渠道打包与签名的关系：**
- V1 签名：META-INF 加文件不影响签名 ✅
- V2 签名：修改 APK Signing Block 中的自定义区域不影响 ✅（Walle 方案）
- 修改 ZIP 内容或 Central Directory → V2 签名失效 ❌

### 延伸问题

- [[多渠道打包方案]]
- [[代码混淆与反编译防护]]
- [[PMS包管理与安装流程]]

## 记忆锚点

V1 = 逐文件签名（慢，有漏洞）。V2 = 整个 APK 分块签名（快，全覆盖）。V3 = V2 + 密钥轮换。安装时优先验 V2，V1 是兜底。
