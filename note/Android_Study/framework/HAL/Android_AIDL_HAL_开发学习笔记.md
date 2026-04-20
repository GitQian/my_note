# Android AIDL HAL 学习笔记

## 1. 概述
从 Android 11 开始，Google 引入了 **AIDL HAL** 作为 HIDL 的替代方案。它是 Project Treble 架构演进的重要一步，旨在统一系统服务（System Service）和硬件抽象层（HAL）的通信机制。

### 为什么选择 AIDL HAL？
- **接口统一**：与 Android Framework 服务使用相同的 Binder 机制。
- **稳定性保证**：通过 `libbinder_ndk` 提供稳定的运行时 ABI 接口。
- **开发简化**：语法比 HIDL 更接近原生 Java/C++ AIDL。

---

## 2. 核心组件与目录结构
以 `hello` 模块为例，典型的目录结构如下：

```text
hello/aidl/
├── Android.bp          # 接口定义 (aidl_interface)
├── vendor/hsae/.../    # .aidl 接口文件
└── default/
    ├── Android.bp      # 服务实现编译 (cc_binary)
    ├── Hello.h         # 服务逻辑实现 (BnHello 派生)
    ├── main.cpp        # 服务进程入口
    ├── *.rc            # Init 启动脚本
    └── *.xml           # VINTF Manifest 片段
```

---

## 3. 开发五部曲

### Step 1: 接口定义 (IHello.aidl)
- **语法特点**：
  - 字符串必须使用 `String`（首字母大写）。
  - 方法参数需要指定方向：`in` (输入), `out` (输出), `inout` (双向)。
- **稳定性标志**：
  - `@VintfStability`：标记该接口用于 VINTF（供应商接口）。

### Step 2: 接口模块配置 (Android.bp)
使用 `aidl_interface` 模块：
```blueprint
aidl_interface {
    name: "vendor.hsae.hardware.hello",
    unstable: true,    // 开发阶段建议设为 true，绕过 Frozen 检查
    backend: {
        ndk: { enabled: true }, // HAL 开发必须启用 NDK 后端
    },
}
```

### Step 3: 服务实现 (NDK 后端)
- **实现类**：继承自 `BnHello`（Binder Native）。
- **返回值**：使用 `ndk::ScopedAStatus::ok()` 表示调用成功。
- **服务注册**：
  - 库依赖：`libbinder_ndk`。
  - 方法：`AServiceManager_addService`。
  - 名称规范：`descriptor + "/instance_name"`。

### Step 4: VINTF 声明
VINTF 是系统发现 HAL 的唯一凭证。
- **格式**：`<hal format="aidl">`。
- **位置**：`/vendor/etc/vintf/manifest/`。
- **作用**：解决 System 和 Vendor 的版本兼容性检查。

### Step 5: SELinux 安全策略
HAL 服务的运行需要三处配置：
1. **`te` 文件**：定义域（domain）和访问权限。
2. **`file_contexts`**：为二进制文件分配标签。
3. **`service_contexts`**：为 Binder 服务名称分配标签。

---

## 4. 关键避坑指南 (Troubleshooting)

| 问题 | 原因 | 解决方法 |
| :--- | :--- | :--- |
| **Failed to resolve 'string'** | AIDL 中类型大小写敏感 | 将 `string` 修改为 `String` |
| **stability: must be empty when unstable is true** | unstable 与 stability 冲突 | 设置 `unstable: true` 后删除 `stability: "vintf"` |
| **Check VINTF failed** | XML 格式或版本描述错误 | 检查 `<hal format="aidl">` 标签，unstable 接口通常不需要 `<version>` |
| **Service list 找不到服务** | 进程未启动或注册失败 | 检查 `logcat` 和 `ps -A`，确认服务进程是否崩溃或被 SELinux 拦截 |

---

## 5. 常用命令
- **检查服务注册**：`adb shell service list | grep hello`
- **查看进程**：`adb shell ps -A | grep hello`
- **日志分析**：`adb logcat -s HSAE_HELLO`
- **SELinux 违例检查**：`adb logcat | grep "avc: denied"`

---
*Created at: 2026-04-16*
