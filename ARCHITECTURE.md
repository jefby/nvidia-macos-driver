# NullMoth macOS NVIDIA 驱动 - 软件架构

本文档描述了 NullMoth macOS NVIDIA 驱动的软件架构，这是一种适用于 NVIDIA Turing 及以后显卡（GTX 16、RTX 20/30/40/50 系列）在 Intel Mac 上运行 macOS 15 Sequoia 的 Metal 驱动。

## 概述

该驱动通过分层架构实现了完整的图形堆栈，使 NVIDIA GPU 能够与 macOS 图形框架（Metal、OpenGL、OpenCL、Core Image、Core ML）协同工作：

```
Metal 应用 / WindowServer / Core Image / MPS / OpenGL (Apple 的 GL-on-Metal)
        |
        v
NVMTLDriver.bundle  (Metal 驱动插件)                         [plugin/]
        |  着色器：Apple AIR  -> SPIR-V                            [translator/]
        v
NVK  (Mesa 的 NVIDIA Vulkan 驱动，带有 NAK 编译器)      [nvk/nvk-macos.patch]
        |  SPIR-V -> NVIDIA 机器码，命令缓冲区
        v
NVRM.kext  (NVIDIA 开源 GPU 内核模块，r610，移植到 XNU) [kexts/NVRM]
        |  与 GPU 的 GSP 固件通信
        v
GeForce RTX GPU  (来自 /Users/Shared/nvfw/nvidia/610.57.04 的 GSP 固件)
```

显示和合成以并行方式运行，并带有额外的内核扩展：
- **NVRMFB.kext** - 用于显示输出的帧缓冲区驱动
- **NVRMAGDC.kext** - 用于显示策略的 AppleGraphicsDeviceControl 兼容层
- **NMHDA.kext** - 用于 NVIDIA GPU 音频功能的 HD 音频驱动
- **NMIntelFB.kext** - Intel 帧缓冲区助手（用于混合图形？）

## 组件分解

### 1. 应用层 (`app/`)
面向 macOS 用户的驱动管理应用程序：
- **1401 Mac 应用** - 主应用程序，用于安装、配置和诊断
- **DiagnosticsNative** - 原生诊断模块（Objective-C++）
- **RuntimeCheck** - 系统完整性检查实用程序
- **Resources** - 资产、脚本和 EFI 组件
- **efi-safe** - 用于安全操作的基于 Rust 的 EFI 实用程序

关键文件：
- `app/Sources/main.swift` - 主应用入口点
- `app/DiagnosticsNative/` - 核心诊断功能
- `app/Resources/*` - 安装脚本、EFI 二进制文件和配置

### 2. Metal 驱动插件 (`plugin/`)
实现 MTLDriver 协议以与 Apple 的 Metal 框架进行交互：
- **NVMTLDriver.bundle** - 由 WindowServer/Core Graphics 加载的 GPUBundle
- 处理命令提交、资源管理和 GPU 加速
- 将 Metal API 调用转换为 SPIR-V 着色器

关键组件：
- `NVMTLDevice.m` - 主要设备实现
- `NVMTLVendorCompiler.m` - 供应商特定着色器编译器接口
- `NVMTLEmbedded.m` - 嵌入式资源处理
- `nvmtl_vk.c/h` - Vulkan 接口层
- `nvconv.metal` - 用于转换操作的 Metal 着色器

### 3. 着色器翻译器 (`translator/`)
将 Apple 中间表示（AIR）着色器转换为 SPIR-V：
- 基于 [`metal2vulkan`](https://github.com/KhronosGroup/metal2vulkan)
- LGPL-3.0 许可组件
- 转换着色器代码以供 NVK Vulkan 驱动使用

结构：
- `translator/` - 核心翻译引擎（Rust）
- `wrapper/` - 构建系统和接口代码
- 提供插件中捆绑的 `libnvmtl_translate.dylib`

### 4. Vulkan 后端 (`nvk/`)
NVK - 来自 Mesa 的 NVIDIA Vulkan 驱动，已为 macOS 打补丁：
- 在 NVIDIA 硬件之上实现 Vulkan API
- 使用 NAK 编译器进行 GPU 着色器编译
- 输出命令缓冲区并管理 GPU 内存
- 来自 Mesa 提交 `17ca6174` 的修补版本，带有 macOS 特定更改

关键方面：
- SPIR-V 到 GPU 机器码的翻译
- 向内核驱动提交命令缓冲区
- 内存管理和资源跟踪
- 安装时位于 `/Library/GPUBundles/nvmtl/`

### 5. 内核扩展 (`kexts/`)
NVIDIA 的开源 GPU 内核模块（r610）移植到 XNU（macOS 内核）：

#### 核心模块：
- **NVRM.kext** - NVIDIA 资源管理器（主要 GPU 驱动）
  - 初始化 GPU，管理上下文，处理 GSP 通信
  - 包含核心 GPU 功能（accel、fb、agdc 子系统）
  - 源码位于 `kexts/NVRM/`

- **NVRMFB.kext** - 帧缓冲区驱动
  - 处理显示输出和扫描输出
  - 源码位于 `kexts/NVRMFB/`

- **NVRMAGDC.kext** - AppleGraphicsDeviceControl 兼容层
  - 实现 NVIDIA 和 macOS 之间的显示策略桥接
  - 源码位于 `kexts/NVRMAGDC/`

- **NMHDA.kext** - HD 音频驱动
  - 通过 NVIDIA GPU 提供音频功能
  - 源码位于 `kexts/NMHDA/`

- **NMIntelFB.kext** - Intel 帧缓冲区助手
  - 协助混合图形配置
  - 源码位于 `kexts/NMIntelFB/`

关键架构注意事项：
- 内核扩展加载到 `/Library/Extensions` 并在辅助内核集合中运行
- 通过邮箱接口与 GPU 固件（GSP）通信
- 实现适配 XNU/macOS 的 NVIDIA 开源 GPU 内核模块
- 使用 IOKit 框架进行设备驱动模型

### 6. 构建系统 (`build/`)
编译各种组件的脚本：
- `build_xlate.sh` - 构建着色器翻译器
- `build_plugin.sh` - 构建 Metal 驱动插件
- `build263.sh` - 构建 NVK Vulkan 驱动（修补的 Mesa）
- `accel_build.sh` - 构建加速内核扩展
- `build_agdc.sh` - 构建 AGDC 内核扩展
- `build_plugin.sh` - 构建 Metal 插件束

### 7. 打包 (`package/`)
包含用于创建可分发安装程序包的打包脚本和资源。

### 8. 文档 (`docs/`)
涵盖以下方面的综合文档：
- 发行说明 (`RELEASE-*.md`)
- 卡片支持 (`CARD-SUPPORT.md`)
- 工作原理 (`HOW-IT-WORKS.md`)
- 安装和故障排除指南

## NVRM.kext 与 GSP 固件在 macOS 上的工作原理

这两个部分是整个驱动能在 macOS 上运行的核心。简单说：**macOS 的 NVRM.kext 本质上是 Linux 上 NVIDIA 开源内核模块（r610）移植到 XNU 内核的产物**。

### NVRM.kext 是什么

- **NVRM** = NVIDIA Resource Manager，NVIDIA 传统的内核驱动核心。
- **r610** = NVIDIA 发布的第 610 系列开源内核模块（版本 `610.57.04`），采用双重 BSD/GPL 许可。
- **移植到 XNU** = 把这段为 Linux 内核编写的代码适配到 macOS 的 XNU 内核。

移植过程中必须替换的子系统：

```
Linux NVRM（内核驱动）      →      macOS NVRM.kext（XNU 内核扩展）
        │                                  ├── I/O 接口：Linux file_operations → IOKit
        │                                  ├── 内存管理：Linux kmalloc/vmalloc → XNU 虚拟内存
        │                                  ├── 中断处理：Linux IRQ 机制 → XNU 中断子系统
        │                                  ├── 设备模型：Linux device file → IOKit 设备树
        │                                  └── GSP 通信：邮箱协议实现
```

NVRM.kext 的主要职责：
- 初始化 GPU 硬件和 GSP 固件
- 管理 GPU 内存和上下文
- 接收上层（NVK）提交的命令缓冲区
- 通过邮箱接口与 GSP 固件通信

### GSP 固件是什么

- **GSP** = Graphics Synthesis Platform，运行在 NVIDIA GPU 内部独立微控制器上的固件。
- 位于 `/Users/Shared/nvfw/nvidia/610.57.04/`，版本必须与 NVRM.kext 匹配。
- 负责图形管线配置、计算任务调度、显示输出、电源状态管理等大部分 GPU 实际工作。

### 为什么能在 macOS 下工作

```
macOS 应用 → Metal 框架 → NVMTLDriver.bundle（用户态）
                              ↓  AIR 着色器 → 翻译器 → SPIR-V
                            NVK（用户态）→ 命令缓冲区
                              ↓
                         NVRM.kext（内核态，Linux 代码移植版）
                              ↓  邮箱 / 共享内存 / 中断
                         GPU GSP 固件
                              ↓
                         GeForce RTX 硬件
```

关键成功因素：

1. **NVIDIA 官方开源代码**：r610 模块是官方公开发布的，提供了成熟的驱动框架，不必从零重写。
2. **社区移植适配**：NullMoth 等社区完成了 XNU 内核适配，把 Linux 的 I/O、内存、中断模型逐一替换为 macOS 对应机制。
3. **固件版本匹配**：NVRM.kext 与 GSP 固件必须同为 `610.57.04`，两者通过固定的邮箱协议通信。
4. **引导配置支持**：OpenCore 配置允许加载未签名内核扩展（`csr-active-config`、`boot-args`、阻止 `IONDRVSupport` 等）。
5. **硬件平台限制**：仅限 Intel Mac + Turing 及以后 GPU，Apple Silicon 无法使用该方案。

### 通信机制

```
NVRM.kext                          GSP 固件
    │                                  │
    ├──▶ 写寄存器 → 发送命令          │
    ├──▶ 读寄存器 → 接收响应          │
    ├──▶ 共享内存区域                 │
    └──▶ 中断 → 事件通知             │
```

一次典型的提交路径：

1. Metal 应用提交渲染命令。
2. `NVMTLDriver.bundle` 转换并交给 NVK。
3. NVK 编译生成 GPU 可执行代码，构建命令缓冲区。
4. NVK 把命令缓冲区提交给 NVRM.kext。
5. NVRM.kext 通过邮箱接口把命令发给 GSP。
6. GSP 执行并返回结果，驱动通知应用完成。

### 一句话总结

```
macOS NVRM.kext = Linux NVRM 内核代码 + XNU 内核适配层
```

这是「NVIDIA 官方开源 + 社区移植」的组合：使用的是同一套经过验证的内核驱动逻辑，只是把它移植到 macOS 内核，并搭配匹配版本的 GSP 固件。

## 数据流

1. **应用提交**：
   - Metal 应用通过 MTLCommandQueue 提交命令缓冲区
   - WindowServer/合成器也提交 OpenGL/Core Image 命令

2. **Metal 插件处理**：
   - `NVMTLDriver.bundle` 拦截 Metal 命令
   - 验证资源，准备命令提交
   - 将着色器编译委托给翻译器

3. **着色器翻译**：
   - 来自 Metal 的 AIR 着色器 → 通过翻译器转换为 SPIR-V
   - 在管道创建时发生（非运行时）

4. **Vulkan 后端**：
   - NVK 接收 SPIR-V，通过 NAK 编译为 GPU 机器码
   - 为 GPU 执行构建命令缓冲区
   - 管理内存分配和资源绑定

5. **内核驱动**：
   - NVRM.kext 从 NVK 接收命令缓冲区
   - 通过寄存器和 GSP 固件编程 GPU
   - 处理内存管理、调度和上下文切换
   - 与 NVRMFB 协调以进行显示输出

6. **显示管道**（并行）：
   - WindowServer 将内容渲染到表面
   - NVRMFB.kext 处理扫描输出到显示端口
   - NVRMAGDC.kext 管理显示策略和电源状态
   - GPU 通过显示连接器输出视频信号

## 安装和运行时

### 安装位置：
- `/Library/GPUBundles/NVMTLDriver.bundle` - Metal 插件
- `/Library/GPUBundles/nvmtl/` - NVK Vulkan 驱动
- `/Library/Extensions/NVRM.kext` - GPU 驱动
- `/Library/Extensions/NVRMFB.kext` - 帧缓冲区
- `/Library/Extensions/NVRMAGDC.kext` - 显示策略
- `/Library/Extensions/NMHDA.kext` - HD 音频（如果存在）
- `/Library/Extensions/NMIntelFB.kext` - Intel 帧缓冲区助手（如果存在）
- `/Users/Shared/nvfw/nvidia/610.57.04/` - GSP 固件

### 运行时要求：
- **启动参数**：`nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0`
- **csr-active-config**：`<430A0000>`（允许无签名内核扩展）
- **OpenCore quirks**：`ResizeGpuBars=13`，`ResizeAppleGpuBars=-1`，阻止 `IONDRVSupport`
- **内核扩展**：安装后必须在安全设置中允许

## 依赖和外部组件

1. **Apple 框架**：
   - Metal，IOKit，Foundation（通过插件/应用）
   - OpenGL，OpenCL，Core Image，Core ML（通过系统框架）

2. **第三方组件**：
   - metal2vulkan（翻译器基础）- LGPL-3.0
   - Mesa NVK 驱动 - MIT（带有 macOS 补丁）
   - NVIDIA 开源 GPU 内核模块（r610）- 双 BSD/GPL
   - GSP 固件 - NVIDIA 可再分发许可证

3. **构建依赖**：
   - Xcode 16+，Rust（稳定版），Meson/Ninja
   - NVIDIA 的开源 GPU 内核模块 @ `610.57.04`
   - 用于 kext 开发的 macOS SDK

## 安全考虑

- 以内核特权运行（内核扩展） - 需要 SIP 修改
- Metal 插件在用户空间运行（WindowServer/上下文）
- 核心驱动组件没有网络暴露
- 崩溃报告需要手动用户启动
- 固件二进制 blob（GSP）是由供应商提供且未经验证的

## 可扩展点

1. **插件层**：可以支持其他图形 API（直接 Vulkan 等）
2. **翻译器**：可以针对其他中间表示
3. **NVK**：可以更新为较新的 Mesa 版本
4. **内核扩展**：可以集成较新的 NVIDIA 内核模块特性
5. **应用程序**：可以添加更多诊断/监控功能

## 限制和已知约束

- 仅支持 macOS 15 Sequoia（XNU 版本兼容性）
- 仅支持 Intel Mac（由于内核扩展要求，不支持 Apple Silicon）
- 仅支持 Turing 及以后的 NVIDIA GPU（不支持 Pascal/更旧的架构）
- 需要具有特定配置的 OpenCore 引导加载程序
- GSP 固件版本锁定为 610.57.04
- 电源管理仅限于基本实现
- 多 GPU 支持有限（单 GPU 主要）

## 未来改进

1. 更新到较新的 GSP 固件版本
2. 通过 nouveau 兼容性添对旧版 NVIDIA 架构的支持
3. 改进电源管理和性能监控
4. 添加直接 Vulkan 支持（绕过 Metal 翻译）
5. 增强多显示和多 GPU 支持
6. 实现更完整的 OpenCL/OpenGL 支持