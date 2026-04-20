# Android 显示流程简述

Android 的显示系统是一个典型的生产者-消费者模型，涉及应用层、系统服务层、硬件抽象层和内核驱动。

## 1. 基本架构模型：生产者与消费者
*   **生产者 (Producer)**: 主要是 App 进程，负责绘制 UI 内容到 Buffer（图形缓冲区）。
*   **消费者 (Consumer)**: 主要是 **SurfaceFlinger**，负责将多个 App 的 Buffer 进行合成并输出。
*   **中转站 (BufferQueue)**: 生产者和消费者之间通过 `BufferQueue` 传递数据。

## 2. 关键组件
*   **Choreographer (编舞者)**: 协调动画、输入和绘制的节奏，接收来自系统的 VSync 信号，触发下一帧的绘制。
*   **Surface**: 对应 App 的一个窗口，内部持有一个 `Canvas` 或 `OpenGL/Vulkan` 上下文。
*   **SurfaceFlinger**: 系统级进程，负责合成所有窗口的 Surface。
*   **HWC (Hardware Composer)**: 硬件合成器，利用硬件加速（如 MDP/DPU）减轻 GPU 的合成负担。

## 3. 核心流程步骤

### 第一阶段：App 端绘制 (生产者)
1.  **VSync 信号**: 系统定时发送 VSync 信号给 App。
2.  **Choreographer 响应**: App 收到信号后，执行 `doFrame`。
3.  **测量/布局/绘制**: 执行 `View.onMeasure` -> `onLayout` -> `onDraw`。
4.  **Dequeue Buffer**: App 从 `BufferQueue` 申请一个可用的 Buffer。
5.  **渲染**: 通过 CPU (Skia/Canvas) 或 GPU (OpenGL) 将 UI 数据写入 Buffer。
6.  **Queue Buffer**: 绘制完成后，App 将 Buffer 还给 `BufferQueue`。

### 第二阶段：SurfaceFlinger 合成 (消费者)
1.  **VSync-sf 信号**: SurfaceFlinger 收到专门的 VSync 信号。
2.  **Acquire Buffer**: SurfaceFlinger 从所有的 `BufferQueue` 中取出最新的 Buffer。
3.  **合成策略**: 
    *   **GPU 合成 (Client Composition)**: 使用 OpenGL/Vulkan 将多个 Buffer 画到一个大 Buffer 上。
    *   **HWC 合成 (Device Composition)**: 直接告诉硬件，这几个 Buffer 分别放在屏幕的什么位置，由硬件直接叠加显示。
4.  **输出**: 合成后的帧数据发送到屏幕控制器（Display Controller）。

### 第三阶段：屏幕显示
1.  **Display Driver**: 内核驱动将数据最终推送到物理屏幕面板。
2.  **刷新**: 屏幕按照其刷新率（如 60Hz, 120Hz）将像素显示出来。

---

### 总结流程图
`VSync` -> `App 绘制 (Queue Buffer)` -> `SurfaceFlinger 合成` -> `HWC/GPU` -> `Display Driver` -> `Screen`
