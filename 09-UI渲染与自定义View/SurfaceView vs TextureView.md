---
tags:
  - UI渲染
  - SurfaceView
  - TextureView
  - 视频
  - 相机
difficulty: A
frequency: medium
---

# SurfaceView vs TextureView

## 问题

"SurfaceView 和 TextureView 有什么区别？为什么视频播放和相机预览常用 SurfaceView？两者各自的适用场景和性能特点是什么？"

## 核心答案（30秒版本）

SurfaceView 拥有独立的 Surface（独立窗口层），由 SurfaceFlinger 直接合成，绘制在独立线程进行，不阻塞主线程；缺点是不能做变换（旋转/缩放/透明）、Z 序固定、无法嵌入 ListView/ScrollView。TextureView 共享宿主 View 的 Surface（同一窗口层），可以像普通 View 一样做变换和动画，但必须在硬件加速下使用，绘制走主线程，性能比 SurfaceView 差。视频/相机/游戏对帧率敏感优先 SurfaceView，需要变换或动画时用 TextureView。

## 深入解析

### 原理层（源码级）

**窗口层级对比：**

```
普通 View 渲染路径：
App → DecorView → ViewRootImpl → Surface → SurfaceFlinger → 显示

SurfaceView 渲染路径：
App → DecorView（透明 hole）→ Surface → SurfaceFlinger → 显示
              ↓
       SurfaceView 独立 Surface → SurfaceFlinger → 直接合成

TextureView 渲染路径：
App → DecorView → ViewRootImpl 的 Surface（共享）→ SurfaceFlinger → 显示
       ↑
  TextureView 通过 OpenGL 纹理写入此 Surface
```

**SurfaceView 核心源码：**

```java
// SurfaceView.java
public class SurfaceView extends View {
    final Surface mSurface = new Surface(); // 独立 Surface
    SurfaceControl mSurfaceControl;
    
    @Override
    protected void dispatchDraw(Canvas canvas) {
        // 在 ViewRootImpl 的 Surface 上挖一个透明洞
        if (mDrawFinalFrame) {
            // 不绘制内容
        }
    }
    
    public SurfaceHolder getHolder() {
        return mSurfaceHolder;
    }
    
    private final SurfaceHolder mSurfaceHolder = new SurfaceHolder() {
        @Override
        public Canvas lockCanvas() {
            // 获取 Canvas 进行绘制（独立 Surface，不影响 UI 线程）
            return mSurface.lockCanvas(null);
        }
        
        @Override
        public void unlockCanvasAndPost(Canvas canvas) {
            // 提交绘制结果到 SurfaceFlinger
            mSurface.unlockCanvasAndPost(canvas);
        }
    };
}
```

**SurfaceView 双缓冲机制：**

```
Front Buffer（显示中）  ←─ SurfaceFlinger 读取
Back Buffer（绘制中）   ←─ App 绘制
                ↓
        交换（swap）
```

**TextureView 核心源码：**

```java
// TextureView.java
public class TextureView extends View {
    private SurfaceTexture mSurface;
    private HardwareLayer mLayer;
    
    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        if (!isHardwareAccelerated()) {
            Log.w(LOG_TAG, "TextureView 必须在硬件加速下使用");
            return;
        }
    }
    
    @Override
    protected void draw(Canvas canvas) {
        // 关键：通过硬件层（OpenGL 纹理）绘制
        if (canvas.isHardwareAccelerated()) {
            HardwareCanvas hardwareCanvas = (HardwareCanvas) canvas;
            applyUpdate();
            applyTransformMatrix();
            hardwareCanvas.drawHardwareLayer(mLayer, 0, 0, mLayerPaint);
        }
    }
    
    public SurfaceTexture getSurfaceTexture() {
        return mSurface;
    }
}
```

**SurfaceTexture 与外部纹理：**

```java
// 相机/视频解码器输出 → SurfaceTexture → OpenGL 外部纹理 → 渲染到 TextureView
SurfaceTexture surfaceTexture = textureView.getSurfaceTexture();
Surface surface = new Surface(surfaceTexture);
mediaPlayer.setSurface(surface); // 视频解码器写入
```

**对比总结：**

| 维度 | SurfaceView | TextureView |
|---|---|---|
| Surface | 独立 Surface | 共享 ViewRootImpl 的 Surface |
| 绘制线程 | 可独立线程（不阻塞主线程） | 必须主线程 |
| 硬件加速 | 不依赖 | 必须开启 |
| 变换支持 | 不支持（旋转/缩放/透明） | 支持（通过 setTransform） |
| 嵌入 ScrollView | 困难（Z 序问题） | 支持 |
| 截图 | 困难（getDrawingCache 失效） | 简单（getBitmap()） |
| 性能 | 高（直接合成） | 较低（多一次纹理拷贝） |
| 内存 | 独立 Buffer | 共享 |
| 可见性切换 | 频繁 attach/detach 开销大 | 低 |

### 实战层（项目经验）

**SurfaceView 视频播放：**

```kotlin
class VideoPlayerActivity : AppCompatActivity(), SurfaceHolder.Callback {
    private lateinit var surfaceView: SurfaceView
    private lateinit var mediaPlayer: MediaPlayer
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        surfaceView = findViewById(R.id.surfaceView)
        // 注册回调（Surface 创建后才能开始播放）
        surfaceView.holder.addCallback(this)
    }
    
    override fun surfaceCreated(holder: SurfaceHolder) {
        mediaPlayer = MediaPlayer().apply {
            setDataSource(videoUrl)
            setDisplay(holder) // 视频帧直接写入 Surface
            prepareAsync()
            setOnPreparedListener { start() }
        }
    }
    
    override fun surfaceDestroyed(holder: SurfaceHolder) {
        mediaPlayer.release()
    }
    
    override fun surfaceChanged(holder: SurfaceHolder, format: Int, width: Int, height: Int) {}
}
```

**SurfaceView 自定义绘制（游戏循环）：**

```kotlin
class GameSurfaceView(context: Context) : SurfaceView(context), SurfaceHolder.Callback {
    private var renderThread: RenderThread? = null
    
    init {
        holder.addCallback(this)
    }
    
    override fun surfaceCreated(holder: SurfaceHolder) {
        renderThread = RenderThread(holder).apply { start() }
    }
    
    override fun surfaceDestroyed(holder: SurfaceHolder) {
        renderThread?.running = false
        renderThread?.join()
    }
    
    private class RenderThread(val holder: SurfaceHolder) : Thread() {
        var running = true
        
        override fun run() {
            while (running) {
                val canvas = holder.lockCanvas() ?: continue
                try {
                    // 在独立线程绘制，不阻塞主线程
                    drawGame(canvas)
                } finally {
                    holder.unlockCanvasAndPost(canvas)
                }
                // 控制帧率
                sleep(16)
            }
        }
    }
}
```

**TextureView 实现视频旋转/缩放：**

```kotlin
class RotatableTextureView(context: Context) : TextureView(context), 
    TextureView.SurfaceTextureListener {
    
    init {
        surfaceTextureListener = this
    }
    
    fun rotateVideo(degrees: Float) {
        // SurfaceView 做不到，TextureView 可以
        val matrix = Matrix()
        matrix.postRotate(degrees, width / 2f, height / 2f)
        setTransform(matrix)
        invalidate()
    }
    
    fun captureFrame(): Bitmap? {
        // 直接获取当前帧 Bitmap（SurfaceView 无法做到）
        return bitmap
    }
}
```

**SurfaceView 黑边问题（Android 7.0+ 解决）：**

```kotlin
// 旧版本切换 Activity 时 SurfaceView 会闪黑
// Android 7.0+ 引入 SurfaceView.setZOrderOnTop / Z 序优化

// 透明 SurfaceView（API 26+）
surfaceView.setZOrderOnTop(true)
surfaceView.holder.setFormat(PixelFormat.TRANSLUCENT)
```

**SurfaceControl 与现代方案（Android 10+）：**

```kotlin
// SurfaceView 配合 SurfaceControl 实现独立子层级（API 29+）
val surfaceControl = SurfaceControl.Builder()
    .setName("CustomSurface")
    .setBufferSize(1920, 1080)
    .build()

SurfaceControl.Transaction()
    .setLayer(surfaceControl, 1)
    .setVisibility(surfaceControl, true)
    .apply()
```

**选型决策树：**

```
需要变换/动画/嵌入 ScrollView？
  ├─ 是 → TextureView
  └─ 否 → 主线程是否繁忙？
           ├─ 是（频繁绘制）→ SurfaceView（独立线程）
           └─ 否 → 普通 View 即可
```

**典型场景：**

| 场景 | 推荐 | 理由 |
|---|---|---|
| 视频播放器（全屏） | SurfaceView | 性能好，主线程不阻塞 |
| 视频列表（嵌入 RecyclerView） | TextureView | 支持滚动复用 |
| 短视频（卡片+缩放） | TextureView | 需要动画变换 |
| 相机预览 | SurfaceView | 低延迟，高帧率 |
| 游戏 | SurfaceView + 渲染线程 | 独立线程绘制 |
| AR/特效 | TextureView | 需要与其他 View 叠加 |

### 延伸问题

- [[View绘制三大流程]] — SurfaceView 如何绕过 ViewRootImpl 的绘制流程
- [[属性动画原理与插值器]] — TextureView 动画与 SurfaceView 动画的实现差异
- [[Compose重组原理与性能优化]] — Compose 中如何嵌入 SurfaceView/TextureView（AndroidView）
- [[事件分发机制]] — SurfaceView 的事件分发特殊性

## 记忆锚点

SurfaceView 独立 Surface 走独立线程，性能高但不能变换；TextureView 共享 Surface 走主线程，能动画但有性能损失。
