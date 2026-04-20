# Android Native Service 编写指南

本指南介绍了如何在 Android 环境下编写、配置并编译一个基于 Binder 机制的原生服务（Native Service）。

## 核心步骤

### 1. 定义接口 (AIDL)
使用 AIDL (Android Interface Definition Language) 定义服务提供的接口。
*   **文件路径**：建议物理路径与 `package` 声明一致。例如 `package com.example;` 对应 `com/example/IExample.aidl`。
*   **注意**：接口名建议以 `I` 开头。

### 2. 实现服务类 (Implementation)
你需要创建一个类来继承由 AIDL 自动生成的基类 `BnExample`（Bn 代表 Binder Native）。
*   **头文件**：引用 `<com/example/BnExample.h>`。
*   **实现逻辑**：重写 AIDL 中定义的虚函数。
*   **返回值**：使用 `::android::binder::Status::ok()` 表示调用成功。

### 3. 编写程序入口 (Main)
在 `main` 函数中完成以下任务：
1.  **初始化 ProcessState**：`ProcessState::self()`。
2.  **获取 ServiceManager**：`defaultServiceManager()`。
3.  **注册服务**：使用 `sm->addService(String16("service_name"), new MyService())`。
4.  **启动线程池**：`ProcessState::self()->startThreadPool()`。
5.  **进入循环**：`IPCThreadState::self()->joinThreadPool()`。

### 4. 配置编译脚本 (Android.bp)
在 `cc_binary` 模块中：
*   **srcs**: 包含生成的 `aidl-gen/*.cpp`。
*   **local_include_dirs**: 包含 `aidl-gen` 以便编译器找到头文件。
*   **shared_libs**: 包含 `libbinder`, `libutils`, `liblog`, `libbase`。
*   **vendor**: 设置为 `true`。

---

## VSCode 支持与手动生成代码 (可选)

虽然 Android 编译系统会自动处理 AIDL，但为了让 VSCode 能够识别头文件并支持 **代码跳转** 和 **自动补全**，建议采用手动生成代码并存放在 `aidl-gen` 目录下的方案（参考 `diagnostic_server` 的做法）。

### 1. 运行生成脚本
在 Android 源码环境下，进入模块目录并运行脚本：
```bash
chmod +x aidl/generate_cpp.sh
./aidl/generate_cpp.sh
```
该脚本会调用 `aidl-cpp` 工具，将 `.aidl` 文件编译成对应的 `.cpp` 和 `.h` 文件，并存放到 `aidl-gen/` 目录下。

### 2. 目录结构说明
*   **aidl/**: 存放原始 AIDL 定义。
*   **aidl-gen/**: 存放生成的 C++ 代码。VSCode 将通过扫描该目录实现 IntelliSense。
*   **aidl-gen/com/example/**: 生成的头文件（如 `BnExample.h`）存放于此。

---

## 系统集成与权限配置 (关键)

### 1. SELinux 策略 (sepolicy)
为了使服务能够自动启动，必须在 `sepolicy` 目录下配置以下文件：
*   **example_service.te**: 定义进程域和 `init_daemon_domain` 转换。
*   **file_contexts**: 将 `/vendor/bin/example_service` 映射到 `example_service_exec` 标签。
*   **service_contexts**: 定义 Binder 服务名对应的标签。

### 2. 修改 BoardConfig.mk
让系统编译系统包含您的 SELinux 策略目录：
```makefile
BOARD_VENDOR_SEPOLICY_DIRS += vendor/hsae/services/example_service/sepolicy
```

### 3. 修改 Product Makefile
确保服务被包含在系统镜像中：
```makefile
PRODUCT_PACKAGES += \
    example_service
```

---

## 常见问题 FAQ

### Q: 为什么手动生成的代码还要放在 `aidl-gen`？
为了 VSCode 的开发体验。自动生成的代码在 `out/` 目录下，路径深且复杂，VSCode 很难准确定位。放在本地 `aidl-gen` 目录下，配合 `.vscode/c_cpp_properties.json` 的 `includePath` 配置，可以实现秒级的代码跳转。

### Q: 服务能手动运行，但开机不启动？
这通常是由于 **SELinux 标签错误** 导致 `init` 拒绝拉起服务。请确认已按照上述步骤配置 `sepolicy` 并重新编译刷入镜像。
