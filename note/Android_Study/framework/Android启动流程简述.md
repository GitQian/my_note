# Android 启动流程简述

Android 系统的启动是一个复杂的分层过程，从硬件上电到用户看到桌面（Launcher），主要经历以下几个关键阶段：

## 1. Loader 层 (Boot ROM & Bootloader)
*   **Power On**: 按下电源键，引导芯片从预定义位置（如 ROM）执行代码。
*   **Bootloader**: 引导加载程序，负责检查 RAM、初始化硬件参数，并最终拉起 Linux 内核。

## 2. Kernel 层 (Linux Kernel)
*   内核启动后会进行硬件自检、内存管理、中断处理等初始化。
*   加载驱动程序（如 Binder、Display、Input 等）。
*   **关键动作**: 内核启动完成后，会在用户空间启动第一个进程 —— **`init` 进程**。

## 3. Native 层 (init 进程)
*   **init 进程**: PID 为 1，是所有进程的父进程。
*   解析 `init.rc` 脚本。
*   启动 `ueventd`（管理设备节点）、`logd`（日志）、`servicemanager`（Binder 服务管理）等核心守护进程。
*   **关键动作**: 启动 **`Zygote`** 进程。

## 4. Java Framework 层 (Zygote & SystemServer)
*   **Zygote 进程**: Android Java 世界的孵化器。
    *   创建虚拟机实例（ART/Dalvik）。
    *   预加载 Java 类库和资源。
    *   监听 Socket 请求。
*   **System Server**: 由 Zygote fork 出来的第一个 Java 进程。
    *   负责启动 Android 核心服务：`ActivityManagerService` (AMS), `WindowManagerService` (WMS), `PackageManagerService` (PMS) 等。
    *   在 ServiceManager 中注册这些服务。

## 5. App 层 (Launcher 启动)
*   AMS 完成系统服务初始化后，会发送 `ACTION_MAIN` 和 `CATEGORY_HOME` 的 Intent。
*   **Launcher**: 桌面程序接收到 Intent 并启动，至此用户看到系统桌面，启动流程完成。

---

### 总结流程图
`Power On` -> `Bootloader` -> `Kernel` -> `init` -> `Zygote` -> `SystemServer` -> `Launcher`
