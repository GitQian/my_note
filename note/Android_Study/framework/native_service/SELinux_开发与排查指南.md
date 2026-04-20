# Android SELinux 开发与实战排查指南

本指南总结了在 Android Treble 架构下，将原生服务（Native Service）从 Vendor 分区迁移至 System_ext 分区时涉及的 SELinux 配置要点及调试技巧。

---

## 1. 核心概念梳理

在执行 `ls -lZ` 或查看日志时，SELinux 上下文（Context）遵循 `user:role:type:s0` 格式：

*   **Domain (域)**：进程的标签（如 `u:r:example_service:s0`）。决定了进程能做什么。
*   **Type (类型)**：文件的标签（如 `u:object_r:example_service_exec:s0`）。决定了谁能访问该文件。
*   **Service Label (服务标签)**：注册到 `servicemanager` 的名称标签。

### 关键属性：`coredomain`
*   **定义**：属于系统核心域的属性。
*   **场景**：任何运行在 `/system` 或 `/system_ext` 分区的进程，如果需要向系统 `servicemanager` 注册服务，**必须**具备 `coredomain` 属性。
*   **后果**：若缺失该属性，会导致 `neverallow` 违规，且进程无法通过 `init` 正常启动。

---

## 2. 策略文件结构

一个标准的 Native Service 需要配置以下三个文件：

### 2.1 `example_service.te` (类型定义与授权)
```te
# 定义进程域并关联 coredomain
type example_service, domain, coredomain;
# 定义执行文件类型
type example_service_exec, exec_type, file_type, system_file_type;

# 关联 init 启动域切换
init_daemon_domain(example_service)

# 授权向 servicemanager 添加服务
add_service(example_service, example_service_service)

# 授权特定 App 发现服务 (精细化管理)
allow { system_app platform_app priv_app shell } example_service_service:service_manager find;
```

### 2.2 `file_contexts` (路径映射)
将物理路径与 SELinux 标签关联。
```te
/system_ext/bin/example_service  u:object_r:example_service_exec:s0
```

### 2.3 `service_contexts` (Binder 服务映射)
将 AIDL 接口名称与服务标签关联。
```te
com.hsae.example.IExample/default  u:object_r:example_service_service:s0
```

---

## 3. 编译系统集成 (.mk)

SELinux 策略必须根据分区正确包含在编译流程中：

*   **Vendor 分区策略**：
    使用 `BOARD_VENDOR_SEPOLICY_DIRS += <路径>`
*   **System_ext 分区策略**（本次迁移的关键）：
    使用 **`SYSTEM_EXT_PRIVATE_SEPOLICY_DIRS += <路径>`**
    *注：使用此变量可确保 `file_contexts` 被正确打包进 `system_ext` 镜像，解决重启后标签丢失的问题。*

---

## 4. 排查“神兵利器”：`ls -lZ`

执行 `ls -lZ /system_ext/bin/example_service` 得到的输出字段含义：

| 字段 | 示例 | 排查要点 |
| :--- | :--- | :--- |
| **标准权限** | `-rwxr-xr-x` | 确认是否有 `x` 执行权限。 |
| **所有者/组**| `root shell` | 确认所有者是否正确。 |
| **SELinux 用户** | `u` | Android 默认为 `u`。 |
| **SELinux 角色** | `object_r` | 文件固定为 `object_r`。 |
| **SELinux 类型** | **`example_service_exec`** | **最核心：必须与定义的 exec 类型一致。** |
| **MLS 等级** | `s0` | 默认等级。 |

---

## 5. 常见问题与坑点总结

### 5.1 "Unrecognized Character" (非法字符报错)
*   **原因**：SELinux 编译器对格式要求极其苛刻。Windows 换行符（CRLF）、缺少结尾换行、不可见字符都会导致整编失败。
*   **修复**：
    ```bash
    # 去除 Windows 换行符
    sed -i 's/\r//g' *.te
    # 确保文件末尾有空行
    echo "" >> example_service.te
    ```

### 5.2 Neverallow 违规
*   **原因**：违反了 Android Treble 的安全红线。例如：
    *   Vendor 域 App 访问 System 域服务。
    *   非 `coredomain` 进程注册系统服务。
*   **修复**：收紧 `allow` 范围，或正确声明 `typeattribute`。

### 5.3 重启后标签失效
*   **原因**：修改了 `file_contexts` 但没有通过正确的 `.mk` 变量包含，导致标签只存在于内存（手动 `chcon`），未持久化到镜像。
*   **修复**：检查 `BOARD_SEPOLICY_DIRS` 是否匹配了文件的实际分区。

---

## 6. 常用调试流程

1.  **确认进程状态**：`ps -eZ | grep <关键词>` (查看进程运行域)
2.  **确认文件标签**：`ls -lZ <路径>` (查看文件执行标签)
3.  **查看拒绝日志**：`dmesg | grep "avc: denied"` 或 `logcat | grep "avc"`
4.  **验证策略加载**：`restorecon -v <路径>` (检查系统是否能根据策略自动恢复标签)
5.  **临时关闭强制模式**：`setenforce 0` (用于排除是否是 SELinux 导致的权限问题)
