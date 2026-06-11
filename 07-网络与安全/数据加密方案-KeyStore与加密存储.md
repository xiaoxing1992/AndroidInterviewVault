---
tags: [android, security, encryption, keystore, data-protection]
difficulty: A
frequency: medium
---

# 数据加密方案 - KeyStore 与加密存储

## 问题

"Android 中如何安全地存储敏感数据？KeyStore 系统是怎么工作的？对称加密和非对称加密怎么选型？EncryptedSharedPreferences 底层原理是什么？"

## 核心答案（30秒版本）

Android KeyStore 系统提供硬件级密钥保护（TEE/StrongBox），密钥生成和使用都在安全硬件内完成，应用无法导出私钥。敏感数据存储推荐使用 Jetpack Security 库的 EncryptedSharedPreferences（底层 AES-256-SIV 加密 key + AES-256-GCM 加密 value）。对称加密（AES）用于大量数据加密，非对称加密（RSA/EC）用于密钥交换和签名。Android 6.0+ 支持 KeyStore 绑定生物认证，实现"解锁才能解密"。

## 深入解析

### 原理层（源码级）

**Android KeyStore 架构：**

```
┌─────────────────────────────────────────┐
│           Application Layer              │
│  KeyStore API / Cipher / Signature       │
├─────────────────────────────────────────┤
│           Keymaster HAL                  │
│  (Hardware Abstraction Layer)            │
├─────────────────────────────────────────┤
│     TEE (TrustZone) / StrongBox         │
│  密钥生成、存储、运算都在此完成           │
│  应用层永远无法获取密钥明文               │
└─────────────────────────────────────────┘
```

**KeyStore 密钥生成源码流程：**

```java
// KeyGenParameterSpec 构建密钥属性
// → KeyGenerator/KeyPairGenerator 生成密钥
// → Keymaster HAL 在 TEE 中执行实际生成
// → 返回密钥引用（非明文）

// AndroidKeyStoreKeyGeneratorSpi.java
@Override
protected SecretKey engineGenerateKey() {
    // 1. 构建 Keymaster 参数
    KeymasterArguments args = new KeymasterArguments();
    args.addEnum(KeymasterDefs.KM_TAG_ALGORITHM, keymasterAlgorithm);
    args.addEnum(KeymasterDefs.KM_TAG_BLOCK_MODE, keymasterBlockMode);
    args.addEnum(KeymasterDefs.KM_TAG_PADDING, keymasterPadding);
    args.addUnsignedInt(KeymasterDefs.KM_TAG_KEY_SIZE, keySize);
    
    // 2. 通过 Binder 调用 KeyStore 服务
    int errorCode = keyStore.generateKey(alias, args, flags);
    
    // 3. 返回密钥引用
    return new AndroidKeyStoreSecretKey(alias, uid);
}
```

**EncryptedSharedPreferences 底层实现：**

```java
// EncryptedSharedPreferences 使用 Tink 加密库
// 加密方案：
// - Key 加密：AES256-SIV（确定性加密，相同 key 产生相同密文，支持查找）
// - Value 加密：AES256-GCM（随机化加密，相同 value 产生不同密文）

// 初始化流程
public static SharedPreferences create(
    String fileName,
    String masterKeyAlias,
    Context context,
    PrefKeyEncryptionScheme prefKeyEncryptionScheme,
    PrefValueEncryptionScheme prefValueEncryptionScheme
) {
    // 1. 获取或创建 Master Key（存储在 KeyStore 中）
    KeysetHandle daeadKeysetHandle = new AndroidKeysetManager.Builder()
        .withKeyTemplate(AesSivKeyManager.aes256SivTemplate())
        .withSharedPref(context, KEY_KEYSET_ALIAS, fileName)
        .withMasterKeyUri(KEYSTORE_PATH_URI + masterKeyAlias)
        .build()
        .getKeysetHandle();
    
    // 2. 创建 AEAD 原语用于 value 加密
    KeysetHandle aeadKeysetHandle = new AndroidKeysetManager.Builder()
        .withKeyTemplate(AesGcmKeyManager.aes256GcmTemplate())
        .withSharedPref(context, VALUE_KEYSET_ALIAS, fileName)
        .withMasterKeyUri(KEYSTORE_PATH_URI + masterKeyAlias)
        .build()
        .getKeysetHandle();
    
    return new EncryptedSharedPreferences(fileName, daeadKeysetHandle, aeadKeysetHandle, context);
}

// 写入时加密
@Override
public Editor putString(String key, String value) {
    // key 使用 DeterministicAead 加密（可查找）
    String encryptedKey = daead.encryptDeterministically(key.getBytes(), fileName.getBytes());
    // value 使用 Aead 加密（随机化）
    byte[] encryptedValue = aead.encrypt(value.getBytes(), encryptedKey.getBytes());
    return delegate.edit().putString(Base64.encode(encryptedKey), Base64.encode(encryptedValue));
}
```

**加密算法选型对比：**

| 场景 | 算法 | 模式 | 说明 |
|------|------|------|------|
| 数据加密 | AES-256 | GCM | 认证加密，防篡改 |
| 密钥确定性加密 | AES-256 | SIV | 相同输入相同输出，支持查找 |
| 密钥交换 | ECDH | P-256 | 比 RSA 更高效 |
| 数字签名 | ECDSA | P-256 | 验证数据完整性 |
| Token 签名 | HMAC | SHA-256 | 对称签名，速度快 |

### 实战层（项目经验）

**EncryptedSharedPreferences 使用：**

```kotlin
// 创建 Master Key
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .setUserAuthenticationRequired(false) // true 则需要生物认证
    .build()

// 创建加密 SP
val encryptedPrefs = EncryptedSharedPreferences.create(
    context,
    "secret_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

// 使用方式与普通 SP 完全一致
encryptedPrefs.edit()
    .putString("auth_token", token)
    .putString("refresh_token", refreshToken)
    .apply()
```

**KeyStore 绑定生物认证：**

```kotlin
// 生成需要生物认证才能使用的密钥
val keyGenSpec = KeyGenParameterSpec.Builder("biometric_key", PURPOSE_ENCRYPT or PURPOSE_DECRYPT)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setKeySize(256)
    .setUserAuthenticationRequired(true)                    // 需要用户认证
    .setUserAuthenticationParameters(30, AUTH_BIOMETRIC_STRONG) // 30秒有效
    .setInvalidatedByBiometricEnrollment(true)             // 新指纹注册后失效
    .build()

val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
keyGenerator.init(keyGenSpec)
keyGenerator.generateKey()

// 使用时需要先通过生物认证
val cipher = Cipher.getInstance("AES/GCM/NoPadding")
cipher.init(Cipher.ENCRYPT_MODE, key)

val biometricPrompt = BiometricPrompt(activity, executor, callback)
biometricPrompt.authenticate(
    BiometricPrompt.PromptInfo.Builder()
        .setTitle("验证身份")
        .setNegativeButtonText("取消")
        .build(),
    BiometricPrompt.CryptoObject(cipher)
)
```

**EncryptedFile 加密文件存储：**

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val encryptedFile = EncryptedFile.Builder(
    context,
    File(context.filesDir, "sensitive_data.bin"),
    masterKey,
    EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
).build()

// 写入
encryptedFile.openFileOutput().use { output ->
    output.write(sensitiveData.toByteArray())
}

// 读取
encryptedFile.openFileInput().use { input ->
    val data = input.readBytes()
}
```

**安全存储方案选型：**

```kotlin
// 决策树
fun chooseStorageStrategy(data: SensitiveData): StorageStrategy {
    return when {
        // 小量键值对（token、密码）→ EncryptedSharedPreferences
        data.type == KeyValue && data.size < 1.MB -> EncryptedSharedPreferences
        
        // 文件数据（证书、配置）→ EncryptedFile
        data.type == File -> EncryptedFile
        
        // 数据库（大量结构化数据）→ SQLCipher
        data.type == Structured && data.size > 1.MB -> SQLCipher
        
        // 临时数据（会话密钥）→ 内存 + 及时清除
        data.lifetime == Session -> InMemoryWithClear
        
        else -> EncryptedSharedPreferences
    }
}
```

**常见安全陷阱：**

```kotlin
// ❌ 错误：硬编码密钥
val key = "my_secret_key_123".toByteArray() // 反编译可见

// ❌ 错误：使用 ECB 模式
Cipher.getInstance("AES/ECB/PKCS5Padding") // 相同明文产生相同密文

// ❌ 错误：IV 重用
cipher.init(Cipher.ENCRYPT_MODE, key, IvParameterSpec(fixedIv)) // GCM 模式 IV 重用会泄露密钥

// ✅ 正确：使用 KeyStore + GCM + 随机 IV
val keyGenerator = KeyGenerator.getInstance("AES", "AndroidKeyStore")
keyGenerator.init(keyGenSpec)
val key = keyGenerator.generateKey()

val cipher = Cipher.getInstance("AES/GCM/NoPadding")
cipher.init(Cipher.ENCRYPT_MODE, key) // GCM 自动生成随机 IV
val iv = cipher.iv // 保存 IV 用于解密
val ciphertext = cipher.doFinal(plaintext)
```

### 延伸问题

- [[HTTPS证书校验与证书锁定]] — 客户端证书的私钥存储在 KeyStore 中
- [[OAuth2-JWT Token刷新机制]] — Token 的安全存储方案
- [[代码混淆与反编译防护]] — 防止密钥硬编码被反编译获取

## 记忆锚点

KeyStore = 硬件级密钥保护（TEE），密钥不出安全区；EncryptedSharedPreferences = AES-SIV 加密 key + AES-GCM 加密 value，Master Key 存 KeyStore；选型口诀：小数据用 ESP，文件用 EF，大数据用 SQLCipher。