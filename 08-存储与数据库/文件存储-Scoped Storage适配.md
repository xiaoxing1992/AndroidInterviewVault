---
tags: [android, 面试, 存储, scoped-storage, A]
difficulty: A
frequency: high
---

# 文件存储 - Scoped Storage 适配

## 问题

> Android 10 引入的 Scoped Storage 是什么？它对文件存储有哪些限制？如何在新旧 API 之间适配？MediaStore 和 Storage Access Framework 各自的使用场景？

## 核心答案

Scoped Storage 限制 App 只能直接访问自己的私有目录和媒体文件，无法随意读写共享存储。访问其他文件需通过 MediaStore（媒体文件）或 Storage Access Framework（用户授权选择文件）。Android 10 可用 `requestLegacyExternalStorage` 临时豁免，Android 11+ 强制启用，必须迁移。

## 深入解析

### 原理层

**存储分类（Android 10+）：**
```
私有目录（无需权限）:
├── /data/data/{package}/                — 内部存储
└── /storage/emulated/0/Android/data/{package}/  — 外部私有

共享目录（需要 MediaStore 或 SAF）:
├── /storage/emulated/0/DCIM/             — 图片
├── /storage/emulated/0/Download/         — 下载
├── /storage/emulated/0/Documents/        — 文档
└── /storage/emulated/0/Music/            — 音乐
```

**MediaStore（媒体文件）：**
```kotlin
// 写入图片到 Pictures 目录
val contentValues = ContentValues().apply {
    put(MediaStore.Images.Media.DISPLAY_NAME, "photo.jpg")
    put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
    put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/MyApp")
}
val uri = contentResolver.insert(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI, 
    contentValues
)
contentResolver.openOutputStream(uri!!)?.use { os ->
    bitmap.compress(Bitmap.CompressFormat.JPEG, 90, os)
}

// 查询所有图片
val cursor = contentResolver.query(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
    arrayOf(MediaStore.Images.Media._ID, MediaStore.Images.Media.DISPLAY_NAME),
    null, null, "${MediaStore.Images.Media.DATE_ADDED} DESC"
)
```

**Storage Access Framework (SAF)：**
```kotlin
// 让用户选择文件
val intent = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
    addCategory(Intent.CATEGORY_OPENABLE)
    type = "application/pdf"
}
startActivityForResult(intent, REQUEST_PICK_FILE)

// 处理结果
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    if (resultCode == RESULT_OK) {
        val uri = data?.data ?: return
        // 持久化权限（重启后仍有效）
        contentResolver.takePersistableUriPermission(
            uri, Intent.FLAG_GRANT_READ_URI_PERMISSION
        )
        contentResolver.openInputStream(uri)?.use { /* 读取 */ }
    }
}
```

**MANAGE_EXTERNAL_STORAGE（最高权限）：**
```xml
<!-- 慎用！需要 Google Play 审核 -->
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
```

### 实战层

**适配清单：**

| 旧 API（弃用） | 新 API |
|---------------|--------|
| `getExternalStorageDirectory()` + 拼路径 | `getExternalFilesDir()` 或 MediaStore |
| `File("/sdcard/...")` | `MediaStore.insert()` |
| 文件路径传递 | URI 传递 |
| `READ_EXTERNAL_STORAGE` | `READ_MEDIA_IMAGES/VIDEO/AUDIO`（API 33+） |

**Android 13+ 媒体权限拆分：**
```xml
<!-- API 33+ 媒体权限细分 -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

<!-- API 34+ 部分访问 -->
<uses-permission android:name="android.permission.READ_MEDIA_VISUAL_USER_SELECTED" />
```

**Photo Picker（Android 13+，推荐）：**
```kotlin
// 系统提供的图片选择器，无需任何权限
val pickMedia = registerForActivityResult(PickVisualMedia()) { uri ->
    if (uri != null) { /* 处理 uri */ }
}
pickMedia.launch(PickVisualMediaRequest(PickVisualMedia.ImageOnly))
```

**坑点：**
- `requestLegacyExternalStorage` 只在 Android 10 有效，Android 11+ 失效
- App 卸载后私有目录数据被清除，重要数据要备份
- `MediaStore.MediaColumns.DATA`（绝对路径）已废弃，必须用 URI

### 延伸问题

- [[ContentProvider跨进程数据共享]]
- [[网络安全配置-Network Security Config]]

## 记忆锚点

Scoped Storage = 沙盒化文件系统。App 只能玩自己的私有目录，要访问别人的文件必须通过 MediaStore（媒体）或 SAF（用户授权）。Android 13+ 媒体权限再细分。
