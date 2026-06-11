---
tags: [android, 面试, 工程化, 字节码, S]
difficulty: S
frequency: medium
---

# Transform API 与字节码插桩

## 问题

> Android 中如何在编译期修改字节码？Transform API 的原理是什么？ASM 和 Javassist 的区别？有哪些实际应用场景？

## 核心答案

Transform API（已废弃，AGP 7.0+ 用 AsmClassVisitorFactory 替代）允许在 dex 打包前拦截所有 class 文件并修改字节码。ASM 是基于访问者模式的字节码操作框架，性能高但需要理解字节码指令；Javassist 基于源码级 API，易用但性能较低。应用场景：无侵入埋点、性能监控、AOP 切面、隐私合规检测。

## 深入解析

### 原理层

**编译流程中的位置：**
```
.kt/.java → kotlinc/javac → .class → Transform/ASM → 修改后的 .class → D8/R8 → .dex
                                      ↑ 这里插入字节码修改
```

**旧 Transform API（AGP < 7.0）：**
```kotlin
class MyTransform : Transform() {
    override fun getName() = "myTransform"
    override fun getInputTypes() = setOf(QualifiedContent.DefaultContentType.CLASSES)
    override fun getScopes() = setOf(QualifiedContent.Scope.PROJECT)
    override fun isIncremental() = true
    
    override fun transform(transformInvocation: TransformInvocation) {
        transformInvocation.inputs.forEach { input ->
            input.directoryInputs.forEach { dir ->
                // 遍历所有 class 文件，用 ASM 修改
                dir.file.walkTopDown().filter { it.extension == "class" }.forEach { file ->
                    val modified = modifyClass(file.readBytes())
                    file.writeBytes(modified)
                }
            }
        }
    }
}
```

**新 API — AsmClassVisitorFactory（AGP 7.0+）：**
```kotlin
// 注册
abstract class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.extensions.getByType<AndroidComponentsExtension>().onVariants { variant ->
            variant.instrumentation.transformClassesWith(
                MyClassVisitorFactory::class.java, InstrumentationScope.ALL
            ) {}
        }
    }
}

// 实现
abstract class MyClassVisitorFactory : AsmClassVisitorFactory<InstrumentationParameters.None> {
    override fun createClassVisitor(
        classContext: ClassContext,
        nextClassVisitor: ClassVisitor
    ): ClassVisitor {
        return MyClassVisitor(nextClassVisitor)
    }
    
    override fun isInstrumentable(classData: ClassData): Boolean {
        return classData.className.startsWith("com.example")
    }
}
```

**ASM 访问者模式：**
```kotlin
class MyClassVisitor(cv: ClassVisitor) : ClassVisitor(Opcodes.ASM9, cv) {
    override fun visitMethod(
        access: Int, name: String, descriptor: String,
        signature: String?, exceptions: Array<String>?
    ): MethodVisitor {
        val mv = super.visitMethod(access, name, descriptor, signature, exceptions)
        if (name == "onClick") {
            return ClickMethodVisitor(mv, name)
        }
        return mv
    }
}

class ClickMethodVisitor(mv: MethodVisitor, private val name: String) : 
    MethodVisitor(Opcodes.ASM9, mv) {
    override fun visitCode() {
        super.visitCode()
        // 在方法开头插入代码: TrackHelper.onMethodEnter("onClick")
        mv.visitLdcInsn(name)
        mv.visitMethodInsn(
            Opcodes.INVOKESTATIC, "com/example/TrackHelper",
            "onMethodEnter", "(Ljava/lang/String;)V", false
        )
    }
}
```

**ASM vs Javassist：**

| 维度 | ASM | Javassist |
|------|-----|-----------|
| API 级别 | 字节码指令级 | 源码级（类似 Java） |
| 性能 | 高（直接操作字节码） | 低（需要编译源码片段） |
| 学习曲线 | 陡（需懂字节码） | 平缓 |
| 增量支持 | 好 | 一般 |
| 使用者 | Gradle Plugin、Hilt | Lancet、部分 AOP 框架 |

### 实战层

**应用场景：**
1. **无侵入埋点**：自动在所有 onClick 方法中插入埋点代码
2. **方法耗时统计**：方法前后插入计时代码
3. **隐私合规**：检测/替换敏感 API 调用（如 getDeviceId）
4. **线程切换检测**：检测主线程调用了耗时方法
5. **日志增强**：自动给所有方法加 Trace 标记

**实际案例 — 方法耗时：**
```kotlin
// 插桩前
fun loadData() {
    // 业务代码
}

// 插桩后（自动生成）
fun loadData() {
    val _start = System.nanoTime()
    try {
        // 业务代码
    } finally {
        val _cost = System.nanoTime() - _start
        MethodTracer.report("loadData", _cost)
    }
}
```

**坑点：**
- 增量编译支持很重要，否则每次全量扫描很慢
- 注意不要修改系统类（android.* / java.*）
- R8 优化可能移除你插入的代码（需要 keep）
- AGP 版本升级经常 break Transform API

### 延伸问题

- [[Gradle构建生命周期与Task依赖]]
- [[热修复原理-类加载与底层替换]]
- [[KSP与KAPT对比]]

## 记忆锚点

字节码插桩 = 在 .class → .dex 之间拦截修改。ASM 是手术刀（精准但要懂解剖），Javassist 是创可贴（简单但粗糙）。新项目用 AsmClassVisitorFactory。
