---
tags: [android, 面试, framework, hwcomposer, surfaceflinger, display, S]
difficulty: S
frequency: medium
---

# HWComposer 详解

## 问题

> HWComposer 是什么？在显示链路中起什么作用？Device 合成和 Client 合成有什么区别？HWC 怎么决定用哪种合成方式？

## 核心答案

HWComposer（Hardware Composer）是 Display HAL 的一部分，运行在 HAL 层，是 SurfaceFlinger 和 Display 硬件之间的桥梁。负责把多个 Layer 合成一个最终画面送到屏幕。合成方式有三种：Device 合成（HWC Overlay，硬件直接叠加，效率高省电）、Client 合成（GPU 渲染到 FrameBuffer，灵活但耗电）、混合合成（部分 HWC 部分 GPU，最常见）。SurfaceFlinger 收集所有 Layer 后调用 HWC.validate() 询问合成方案，HWC 根据硬件能力（Overlay 数量、格式、变换）返回每个 Layer 的合成类型，然后执行 presentDisplay() 送显。HWC 还负责产生硬件 VSYNC 信号。

## 深入解析

### 原理层（源码级）

**HWComposer 在显示链路中的位置：**

```
App 进程                    SurfaceFlinger              HWComposer              Display
──────────                 ────────────────           ─────────────            ───────

绘制内容
  → lockCanvas / GPU
  → queueBuffer
                                 │
                            VSYNC-sf 到来
                                 │
                            acquireBuffer
                            收集所有可见 Layer
                                 │
                            ① 判断每个 Layer 用什么方式合成
                                 │
                    ┌────────────┼────────────┐
                    ↓            ↓            ↓
              Device 合成    混合合成     Client 合成
            (HWComposer)   (部分 HWC    (GPU 合成)
                             部分 GPU)       │
                    │            │        SurfaceFlinger
                    ↓            ↓        用 GPU 渲染
              HWC 直接叠加    部分 HWC     到 FrameBuffer
              多个 Layer     部分 GPU         │
                    │            │            │
                    ↓            ↓            ↓
                    提交 FrameBuffer 给 Display HAL
                                 │
                                 ↓
                            Display 刷新屏幕
```

---

### 三种合成方式

```
① Device 合成（HWC Overlay）
  → Display 硬件直接叠加多个 Layer
  → 不经过 GPU
  → 效率最高、最省电
  → 限制：Layer 数量有限（通常 4-8 个），Layer 有格式/变换限制

② Client 合成（GPU 合成）
  → SurfaceFlinger 用 GPU 把所有 Layer 渲染到一个 FrameBuffer
  → 再把这一个 FrameBuffer 送给 HWC
  → 没有 Layer 数量限制，支持任意变换
  → 但 GPU 开销大，耗电

③ 混合合成
  → 部分 Layer 用 HWC Overlay
  → 部分 Layer 用 GPU 合成
  → 平衡性能和灵活性
  → 最常见的实际场景
```

**HWC 如何决定合成方式：**

```
SurfaceFlinger 收集所有可见 Layer 后：
  → 调用 HWC.validate() 询问 HWC
  → HWC 检查硬件能力：
      → Layer 数量是否超过 Overlay 限制？
      → Layer 格式是否支持（RGBA/YUV 等）？
      → Layer 变换是否支持（旋转/缩放）？
      → Layer 混合模式是否支持？
  → 对每个 Layer 返回合成类型：
      → HWC2_COMPOSITION_DEVICE    → 用 HWC Overlay
      → HWC2_COMPOSITION_CLIENT    → 用 GPU 合成
      → HWC2_COMPOSITION_SOLID_COLOR → 纯色，硬件直接填
      → HWC2_COMPOSITION_CURSOR    → 硬件光标层
  → SurfaceFlinger 根据结果执行
```

---

### HWComposer 核心接口

```cpp
// HWComposer 是 SurfaceFlinger 与 Display HAL 之间的接口层

class HWComposer {
    // ① 注册 Layer
    void setLayerBuffer(displayId, layerId, buffer);

    // ② 设置 Layer 属性
    void setLayerCompositionType(displayId, layerId, type);
    void setLayerDisplayFrame(displayId, layerId, frame);    // 显示区域
    void setLayerSourceCrop(displayId, layerId, crop);       // 源裁剪
    void setLayerTransform(displayId, layerId, transform);   // 旋转/翻转
    void setLayerBlendMode(displayId, layerId, mode);        // 混合模式
    void setLayerVisibleRegion(displayId, layerId, region);  // 可见区域

    // ③ 验证合成方案
    bool validate(displayId);
    // → HWC 检查每个 Layer 能否用 Device 合成
    // → 返回 changed = true 表示某些 Layer 需要改为 Client 合成

    // ④ 执行合成
    bool present(displayId);
    // → HWC 执行合成并送显
    // → 返回 retireFence（上一帧显示完成的信号）

    // ⑤ 获取 VSYNC 信号
    void registerCallback(callback);
    // → HWC 通过 callback 通知 SurfaceFlinger VSYNC 事件
};
```

```
HWC2 接口（HAL 层，HIDL 定义）：

IComposerClient.hal:
  → createLayer()               // 创建 Layer
  → destroyLayer()              // 销毁 Layer
  → setLayerBuffer()            // 设置 Layer 的 Buffer
  → setLayerCompositionType()   // 设置合成类型
  → validateDisplay()           // 验证合成方案
  → presentDisplay()            // 执行合成送显
  → getDisplayCapabilities()    // 查询硬件能力
  → registerCallback()          // 注册 VSYNC/HOTPLUG 回调
```

---

### SurfaceFlinger 调用 HWComposer 的流程

```cpp
// SurfaceFlinger.cpp — 合成一帧的流程
void SurfaceFlinger::handleMessageRefresh() {
    // ① 收集所有可见 Layer
    for (auto& layer : mDrawingState.layersSortedByZ) {
        if (layer->isVisible()) {
            hwc.setLayerBuffer(displayId, layerId, layer->getBuffer());
            hwc.setLayerDisplayFrame(displayId, layerId, layer->getFrame());
            hwc.setLayerSourceCrop(displayId, layerId, layer->getCrop());
            hwc.setLayerTransform(displayId, layerId, layer->getTransform());
            hwc.setLayerCompositionType(displayId, layerId, HWC2_COMPOSITION_DEVICE);
        }
    }

    // ② 让 HWC 验证合成方案
    bool changed = hwc.validateDisplay(displayId);
    if (changed) {
        // 某些 Layer 不能用 Device 合成，需要改为 Client 合成
        // → SurfaceFlinger 用 GPU 渲染这些 Layer
    }

    // ③ 对需要 Client 合成的 Layer，用 GPU 渲染
    if (hasClientComposition) {
        // SurfaceFlinger 调用 OpenGL/Mesk
        // 把这些 Layer 合成到一个 ClientTargetBuffer
        hwc.setClientTarget(displayId, clientTargetBuffer);
    }

    // ④ 执行合成送显
    hwc.presentDisplay(displayId);
    // → HWC 硬件叠加 Device Layer + ClientTargetBuffer
    // → 输出到 Display
    // → 返回 retireFence
}
```

---

### HWComposer 的 VSYNC 机制

```
HWComposer 负责产生 VSYNC 信号：

硬件 VSYNC：
  → Display 控制器在每次扫描完一帧后产生硬件中断
  → HWC 收到中断
  → 通知 SurfaceFlinger

软件 VSYNC（DispSync）：
  → 硬件 VSYNC 有抖动（时钟偏差）
  → DispSync 用软件模型预测后续 VSYNC 时间点
  → 保证 VSYNC 信号稳定

传递链路：
  Display 硬件中断
    → HWC.onVsync()（HAL 层回调）
      → SurfaceFlinger
        → DispSync（软件预测）
          → AppEventThread → VSYNC-app → App 进程
          → SFEventThread  → VSYNC-sf  → SurfaceFlinger 自己
```

```cpp
// HWComposer.cpp
void HWComposer::onVsync(int32_t displayId, nsecs_t timestamp) {
    // HWC 收到硬件 VSYNC 后回调
    if (mCallback) {
        mCallback->onVsyncReceived(displayId, timestamp);
    }
}

// SurfaceFlinger 收到后：
void SurfaceFlinger::onVsyncReceived(int32_t displayId, nsecs_t timestamp) {
    mDispSync->addVsyncTimestamp(timestamp);
    // DispSync 预测后续 VSYNC 时间点
    // 分发给 AppEventThread 和 SFEventThread
}
```

---

### Overlay 能力与限制

```
典型的移动设备 HWC Overlay 能力：

Layer 数量限制：4-8 个（取决于芯片 DPU 能力）
  → 超过限制的 Layer 必须用 GPU 合成

支持的格式：
  → RGBA_8888（最常见）
  → RGB_565
  → YUV（视频播放）
  → UBWC（压缩格式，高通芯片）

支持的变换：
  → 旋转（0/90/180/270 度）
  → 缩放（整数倍/非整数倍）
  → 翻转（水平/垂直）

支持的混合模式：
  → None（不透明）
  → Premultiplied（预乘 Alpha）
  → Coverage（覆盖 Alpha）

不支持的情况 → 回退 Client（GPU）合成：
  → Layer 数量超限
  → 不支持的格式
  → 不支持的变换
  → 复杂混合模式
```

---

### DPU 与 HWComposer 的关系

```
DPU = Display Processing Unit
  → 现代手机芯片中的专用显示硬件
  → 高通叫 DPU/MDSS，联发科叫 DISP/OVL，三星叫 DPU/FIMD

DPU 的能力：
  → 多个 Overlay 管线（Pipe/Plane）
  → 每个管线可以处理一个 Layer
  → 支持缩放、旋转、颜色空间转换
  → 硬件混合多个 Layer
  → 输出到 MIPI DSI / HDMI 等接口

HWC 与 DPU 的关系：
  → HWC 是软件接口（HAL）
  → DPU 是硬件实现
  → HWC 把 Layer 参数转换为 DPU 寄存器配置
  → HWC 调用 DPU 驱动执行合成

架构：
  SurfaceFlinger
      ↓ HIDL/AIDL
  HWComposer HAL
      ↓
  Display 驱动（内核态）
      ↓
  DPU 硬件
      ↓
  Display 面板
```

---

### HWComposer 版本演进

```
HWC 1.0（Android 4.x）
  → 接口简单，一次性提交所有 Layer
  → 不支持 Layer 级别的合成类型选择

HWC 1.4（Android 5.x）
  → 支持 Layer 级别的 composition type
  → 支持 validate + present 两步操作
  → 支持 VSYNC 回调

HWC 2.0（Android 8.x+）
  → HIDL 接口（HAL 接口定义语言）
  → 更细粒度的 Layer 控制
  → 支持色彩管理、HDR 显示、多 Display

HWC 2.4+（Android 10+）
  → 支持亮度控制
  → 支持 Display 颜色转换
  → 支持更多的 VSYNC 模式
```

---

### 实战层（项目经验）

**为什么视频播放用 SurfaceView 更好？**

```
普通 View：
  → 视频帧是一个 Layer，UI 是一个 Layer
  → 至少 2 个 Layer 需要合成
  → 如果 UI 复杂，可能超过 Overlay 限制 → 回退 GPU 合成

SurfaceView：
  → 视频帧是独立的 Surface（独立 Layer）
  → UI 是另一个 Layer
  → 视频 Layer 通常是 YUV 格式 → HWC Overlay 原生支持
  → HWC 直接叠加，不经过 GPU → 省电、性能好
  → 适合高频渲染（60fps 视频、游戏）
```

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| HWComposer 是什么？ | Display HAL 的一部分，SurfaceFlinger 与 Display 硬件之间的桥梁 |
| HWC 做什么？ | 把多个 Layer 合成一个画面送到屏幕，替代 GPU 合成 |
| Device 合成和 Client 合成区别？ | Device 用硬件 Overlay，效率高省电；Client 用 GPU，灵活但耗电 |
| HWC 怎么决定合成方式？ | SF 调用 validate()，HWC 根据硬件能力返回每个 Layer 的合成类型 |
| Overlay 数量限制？ | 通常 4-8 个，取决于芯片 DPU 能力 |
| Layer 超过 Overlay 限制怎么办？ | 超出的 Layer 用 GPU 合成（混合合成） |
| HWC 和 DPU 的关系？ | HWC 是软件接口（HAL），DPU 是硬件实现 |
| HWC 与 VSYNC 的关系？ | HWC 产生硬件 VSYNC 信号，通知 SurfaceFlinger |
| presentDisplay 做了什么？ | HWC 执行硬件合成并输出到 Display，返回 retireFence |
| 什么情况下回退 GPU 合成？ | Layer 超限、格式/变换不支持、复杂混合模式 |

### 延伸问题

- [[WMS窗口管理与Surface体系]] — WMS 窗口管理与 Surface 体系
- [[Surface绘制原理]] — Surface 的 Buffer 绘制与提交
- [[VSYNC详解]] — HWC 产生的 VSYNC 信号传递链路
- [[Activity显示原理]] — Activity 到屏幕的完整显示链路

## 记忆锚点

HWComposer = "硬件投影仪"。SurfaceFlinger 是导演，把所有 Layer（窗口）交给投影仪。投影仪有两种模式：Device 合成（硬件直接叠加，省电）和 Client 合成（GPU 渲染，灵活）。先 validate 问投影仪"你能不能处理"，能的用硬件，不能的用 GPU，最后 present 送显。
