---
tags: [android, 面试, kotlin, 泛型, A]
difficulty: A
frequency: high
---

# 内联函数与 reified 泛型

## 问题

> inline 函数的原理是什么？为什么只有 inline 函数才能使用 reified 泛型？noinline 和 crossinline 分别解决什么问题？

## 核心答案

inline 函数在编译时将函数体直接内联到调用处，消除函数调用开销和 Lambda 对象分配。reified 依赖内联：因为泛型在 JVM 上会被擦除，但内联后泛型类型实参会被替换为具体类型，所以可以在运行时获取类型信息。noinline 阻止某个 Lambda 参数被内联，crossinline 禁止 Lambda 中使用非局部返回。

## 深入解析

### 原理层

**inline 编译过程：**

```kotlin
inline fun <reified T> startActivity(context: Context) {
    context.startActivity(Intent(context, T::class.java))
}

// 调用处
startActivity<DetailActivity>(this)

// 编译后等价于（内联展开）
this.startActivity(Intent(this, DetailActivity::class.java))
```

**为什么 reified 必须 inline：**
- JVM 泛型擦除：`T` 在运行时变成 `Object`，无法获取实际类型
- inline 后函数体被复制到调用处，编译器知道 `T` 的具体类型，直接替换为实际类型字面量
- 所以 `T::class.java` 在编译后变成 `DetailActivity.class`

**noinline 场景：**
```kotlin
inline fun foo(inlined: () -> Unit, noinline notInlined: () -> Unit) {
    // notInlined 可以作为对象传递给其他函数
    bar(notInlined)
}
```
被 inline 的 Lambda 不是对象，无法赋值给变量或传递给非 inline 函数。

**crossinline 场景：**
```kotlin
inline fun createRunnable(crossinline block: () -> Unit): Runnable {
    return Runnable { block() }  // block 在另一个 Lambda 中调用
}
```
crossinline 禁止非局部返回（`return`），因为 block 可能在其他执行上下文中被调用。

### 实战层

- **适合 inline 的场景**：高阶函数（带 Lambda 参数）、reified 泛型需求、性能敏感的工具函数
- **不适合 inline 的场景**：函数体很大（会导致字节码膨胀）、没有 Lambda 参数的普通函数
- **Android 常见用法**：`startActivity<T>()`、`findViewById` 替代方案、JSON 解析 `fromJson<T>()`
- **坑点**：inline 函数不能访问 private 成员（因为代码被内联到调用处，调用处可能没有访问权限）

### 延伸问题

- [[KSP与KAPT对比]]
- [[委托属性原理与自定义委托]]

## 记忆锚点

inline = 复制粘贴函数体到调用处。reified 能用是因为内联后编译器知道 T 是谁，直接替换。
