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

## NVRM.kext 源码级拆解（`kexts/NVRM/`）

该目录约 14.7k 行（66 个文件），是整个项目技术密度最高的部分。一句话概括：**NVRM.kext 是 NVIDIA 官方 Linux 开源内核模块（r610）与 macOS XNU 内核之间的适配壳**。它不自己实现显卡逻辑，而是把 NVIDIA RM（Resource Manager）代码直接编译进来，再补齐 RM 在 Linux 上依赖、macOS 没有的那批 `os_*` 函数。

`nv-xnu.h` 直接引入 NVIDIA 的头文件，说明这是 hard-fork 式移植：

```c
#define NVRM 1
#include "cpuopsys.h"          // NVIDIA 的 OS 抽象接口声明
#include "nv.h"                // RM 主头
#include "os-interface.h"
#include "nv-kernel-rmapi-ops.h"
#undef NVRM
```

链接时使用 `libnvkernel.a` / `libnvkms.a`（`rmcc.py` 中即写明 "kext links it beside libnvkernel.a"）。

### 目录内容分类

| 类别 | 文件 |
|---|---|
| IOKit 粘合（主驱动） | `NVRM.cpp`(1326)、`nv-xnu.h` |
| OS 抽象实现 | `os-xnu.cpp`(301)、`os-xnu2.cpp`(1396) |
| Linux 缺失符号的弱桩 | `nv_xnu_stubs.cpp`(1815) |
| 直连 RM 内部的功能 | `nvrm_gpuva.cpp`、`nvrm_surfshare.cpp`、`nvrm_pageoff.c`、`nvrm_fbinfo.cpp`、`nvrm_kapi_fields.c` |
| KMS / 显示 | `os-nvkms-xnu.cpp`(371)、`nvkms_shaders.cpp`(1381)、`nvidia-modeset-os-interface.h` |
| 子 kext | `accel/`、`fb/`、`agdc/` |
| 构建/补丁 | `build-completion-archive.py`、`rmcc.py`、`patches/610-display-completion.patch` |
| 用户态工具（**不属于 kext**） | `kms.inc`、`push.inc`、`mirror.inc` |

> 层次上的小毛病：`.inc` 这三份是用户态测试程序（用了 `printf`、`IOConnectCallMethod`、`Foundation`/`IOSurface`），却放在 kext 源码目录里。

### NVRM.cpp —— IOKit 粘合层

`NVRM : IOService`，`IOUserClientClass = NVRMUserClient`。

**匹配与启动**
- `probe()`：先按 boot-arg / `nullmoth-remove` 判断驱动是否已被要求关闭（`nvrm_driver_off`），再调 NVIDIA 自己的 `rm_is_supported_pci_device()` 决定认不认这张卡。
  - Info.plist 故意用**宽匹配**（vendor `0x10DE` + class `0x03`），真正的闸门交给 RM——注释写明 "a mask match is a WIDE net ON PURPOSE"。
- `start()`：读 PCI 信息 → `snapshotBootScreen()` 拷贝开机控制台画面 → `placeLargeBar1()` → `readBARs()` → 建 client 表 → 启动 `autoGo` 线程。

**两趟 bring-up（`go(pass)`）**

```
pass 1: rm_init_rm → rm_init_event_locks → rm_is_supported_device
        → rm_init_private_state → rm_set_rm_firmware_requested
pass 2: setBusMasterEnable → 找 MSI → 建 IOWorkLoop + 中断源
        → rm_init_adapter  ← GSP 固件在这一步真正启动
        → 查询 VRAM → nvkms_xnu_load() → 发布 NVRMDisplay nub
```

`rm_init_adapter` 就是那句 "GSP boot from inside macOS"。

**关键机制**

- **Boot hold（启动闸门）**：`start()` 里 `adjustBusy(1)` 把 IORegistry 标记为 busy，挡住 WindowServer 的 `IOKitWaitQuiet`，直到显示栈就绪才 `releaseBootHold`；另有 `bootHoldCap` 定时器（settle + 40s）兜底。解决"显示还没 armed 就被 WindowServer 抢走"的竞态。
- **BAR1 放置 + Resizable BAR**（最硬核的一段）：读 PCIe Resizable BAR capability，若 BAR1 < 4GB 则尝试放大到 4/8GB（16GB 会启动失败，8GB 经硬件验证）；从 ACPI `_CRS` 解析 host bridge 的 64 位内存窗口，用 CPUID `MAXPHYADDR` 裁剪；要求父桥是 **Root Port**（eGPU/Thunderbolt 直接放弃）；检查区间不与任何其他 PCI 设备/桥窗口重叠；写 BAR、改根端口窗口、**回读校验**，不一致就回滚。
- **自动 bring-up 线程 `autoGo`**：延时（默认 10s，BAR1 放不下时 100s）→ `measureBootRaster()`（直接读 GPU 寄存器 `0x616330` 测刷新率）→ `go(2)` → 轮询 `debug.nvaccel_heads_published` → 依次打开 `debug.nvaccelfb` / `nvrmfb_agdc` / `nvaccel_iop` → 释放 boot hold。整套由 sysctl 驱动的状态机。
- **中断**：`IOFilterInterruptEventSource` + MSI，`interruptOccurred` 调 RM 的 `rm_isr` / `rm_isr_bh`，用 `nv_xnu_isr_thread` 标记 ISR 上下文（`os_is_isr()` 依赖它）。

**NVRMUserClient —— 用户态接口**
- client 表 `gNvClients[32768]`，按 pid+fd 查找。
- external methods：`go / esc / getfd / bindfd / kms / kmsevents / waitevent`。
- `esc` 基本是把 ioctl **透传**给 `rm_ioctl`（另有 CARD_INFO / 版本检查 / surface share 特殊处理）。
- `clientMemoryForType` 实现 mmap：控制设备走 `nv_xnu_alloc_md` 子区间，设备端走 `nv_pm_is_gpu` 校验后的物理区间（少数明确的安全检查，防止用户态映射任意内核内存）。
- 事件：自研 `nv_post_event` → `thread_wakeup`，`sWaitEvent` 用 `assert_wait_timeout`。RM 官方事件机制在 XNU 上不存在，因此重写了一套。

### OS 抽象层（移植工作量的主体）

- **`os-xnu.cpp`**：基础原语——内存 `kern_os_malloc`、锁（`IOLock` / `IOSimpleLock` / `IORWLock` / Mach semaphore）、时间 `mach_absolute_time`、PCI handle、`os_map_kernel_space`、in/out 端口（裸 `inb/outb`）、环形日志。`os_get_cpu_count()` **硬编码返回 4**，`os_get_cpu_number()` 返回 0。
- **`os-xnu2.cpp`**（1396 行）：DMA、内核映射、firmware、work item、wait queue，以及一大坨**内存分配追踪**（`nv_xnu_live`、`nv_q_*`、`nv_pm_*`）。后者维护"物理页 → 归属进程/状态"表，检测 stale 映射和泄漏，说明这块曾出现难查的内存 bug。
- **`nv_xnu_stubs.cpp`**（1815 行）：**全是 `__attribute__((weak))` 空桩**，返回失败 / no-op，覆盖 ACPI 方法、Tegra 时钟、I2C、dma-buf、IMP、SMMU、电源等。这是"把 Linux/Tegra 专属依赖整体挖空"的做法，链接时可被真正的实现覆盖。

固件加载在 `os-xnu2.cpp`：

```c
#define NV_FIRMWARE_FOR_NAME(name) "/Users/Shared/nvfw/nvidia/610.57.04/" name ".bin"
// nv_get_firmware(): vnode_open + vn_rdwr 把 gsp_ga10x / gsp_tu10x / ucodes 读进内存
```

只认 GA10x/AD10x/GH100/GB/GR 和 TU10x/TU11x/GA100 两族，版本号硬编码在路径里。

### 直连 RM 内部的功能模块

这些绕过公开接口，直接调 NVIDIA 的 `rm_kernel_rmapi_op()` / 内部结构体：

| 文件 | 作用 |
|---|---|
| `nvrm_gpuva.cpp` | 用 `NV04_ALLOC` 建 `FERMI_VASPACE_A` + `NV50_MEMORY_VIRTUAL`，`NV04_MAP_MEMORY_DMA` 做映射，给 NVK/accel 提供 GPU 虚拟地址 |
| `nvrm_surfshare.cpp` | `NV04_SHARE` 共享对象；page-off 走 CE |
| `nvrm_pageoff.c` | **直接用 RM 内部 `memdesc*` / `ceutilsMemcopy`**，把 GPU 显存内容拷到一组物理页（合成器取帧用）；带 run 合并优化和 sysctl 统计 |
| `nvrm_fbinfo.cpp` | 建 root client → `GPU_ATTACH_IDS` → `DEVICE_0` → `SUBDEVICE_0` → `FB_GET_INFO_V2` 查 VRAM |
| `nvrm_kapi_fields.c` | 读取 `NvKmsKapiDevice/Memory` 里的 RM handle |

`rmcc.py` 证实了这点：它从 NVIDIA 自己的 `compile_cmds.sh` 里抠出编译 `osapi.c` 的完整命令行，替换源文件来编译这些"内部文件"——必须与 RM 看到的结构体布局完全一致。

### 显示（KMS）路径

- `os-nvkms-xnu.cpp`：`nvKmsModuleLoad` / `nvKmsOpen/Close/Ioctl`、NVKMS 定时器队列、每 open 的 `nvkms_per_open`、内核态事件回调。
- `nvkms_shaders.cpp`：**内嵌 NVIDIA `nvidia-modeset` 的 xz 压缩二进制 shader blob**（cursor/合成用）。
- `patches/610-display-completion.patch`：给 `libnvkms.a` 打补丁，新增 `getFlipPendingStatus`（查询翻转是否完成），并用 `nvKmsResetNotifier` 重置 notifier 槽位；`build-completion-archive.py` 只替换 completion 对象后重打包 `libnvkms.a`。

### 三个子 kext

- **`accel/`（NVAccel）**：实现 `IOGraphicsAccelerator2`，WindowServer 合成走这里。Info.plist 声明 `MetalPluginName = /Library/GPUBundles/NVMTLDriver`、`MetalPluginClassName = NVMTLDevice`——这是 Metal 插件与内核加速器的连接点。`tahoe_fwd.h`（"macOS 26：继承 Apple 不再导出的槽位"）说明在适配新系统 vtable 变化。
- **`fb/`（NVRMFB）**：`NVRMNVDAFramebuffer : IOFramebuffer`，provider 是 `NVRMDisplay` nub；另有 `NVRMFBClaim`（probe 30000）**挤掉 Apple 的 `IONDRVFramebuffer`**。
- **`agdc/`（NVRMAGDC）**：`AppleGraphicsDeviceControl` 兼容层（`NVDAGDCLoader` on IOResources），用 `#pragma pack` + `static_assert` 严格对齐 Apple 私有结构体 `NVAGDCVendorInfo(0x2c)`、`NVAGDCGPUCapability(0xdc)`。

### 代码质量与风险

**做得扎实的地方**
- BAR1 放置有完整的"先验证、写、回读、失败回滚"流程。
- 内存映射有 `nv_pm_is_gpu` 白名单，避免用户态映射内核内存。
- 大量 weak 桩让 Linux 专属依赖优雅降级。
- 错误路径普遍有日志和状态回写。

**脆弱 / 风险点**
1. **深度耦合 NVIDIA 内部未稳定接口**（`rm_kernel_rmapi_op`、`memdesc*`、`ceutils*`、`NvKmsKapi` 内部字段）→ 必须锁死 `610.57.04`，换版本几乎必然重编/重打补丁。
2. **硬编码寄存器偏移**：`measureBootRaster` 的 `0x616330`、`0x682064`，`nvrm_fbinfo.cpp` 的 class id `0x90f1`/`0x50a0` 等，均为逐芯片推断的魔法值。
3. **硬编码常量**：固件路径版本号、`os_get_cpu_count()=4`。
4. **安全面**：kext 无签名（需关 SIP），userclient 的 `esc` 直接透传 ioctl 给 RM，attack surface 很大；`os_check_access()` 直接 `return NV_TRUE`。
5. **调试代码密度极高**：`kprintf`、sysctl、`nv_xnu_chat()` 开关、live alloc dump、pmap owner 表——典型的逆向/调试驱动形态，RELEASE 版才会剥掉。

## 硬件拓扑

驱动面对的不是「接口类型（PCIe / Thunderbolt / USB-C）」，而是四层硬件拓扑。以下每层都标注源码依据。

### 1. 主机侧 PCIe 拓扑

**原生（源码按此实现，也是 RTX 5060 实测场景）**

```
CPU（含 Root Complex）
  ├─ DMI ── PCH ── PCIe Root Port ──┐
  └─ 或 CPU 直连 PCIe lane ── Root Port ──┐
                                          │ PCIe x16
                                   ┌──────▼──────┐
                                   │  NVIDIA GPU │
                                   └─────────────┘
```

**Thunderbolt / eGPU**

```
CPU ── Root Complex ── Root Port
                         └─ Thunderbolt 控制器（NHI，PCIe 设备）
                             └─ ~~~ TB 线缆 / USB-C 口 ~~~
                                 └─ TB 外设内部 Downstream Port
                                     └─ NVIDIA GPU
```

源码判断点（`kexts/NVRM/NVRM.cpp:760-771`）：从 GPU 沿 IORegistry 往上找**直接父桥**，读其 PCIe capability：

- `port type == 4`（Root Port）→ 原生，允许重排 BAR1；
- `port type != 4`（Thunderbolt / PCIe switch 的下游端口）→ 拒绝搬 BAR1。

IOKit 中一个 PCIe 桥是两层对象，所以代码要向上找两层：

```
GPU（IOPCIDevice，NVRM 绑定这里）
 └─ IOPCI2PCIBridge      ← fPCI->getProvider()（总线 nub）
     └─ 父桥 IOPCIDevice  ← pp->getProvider()，检查其是否为 Root Port
```

> 纯 USB-C（无 Thunderbolt 隧道）不构成 PCIe 通道，显卡不会出现在 PCI 总线上。

### 2. 显卡侧 GPU 内部拓扑

源码通过 BAR0 直接读寄存器（`NVRM.cpp::nv_xnu_dump_gsp_regs`）：

```
┌──────────────────────── NVIDIA GPU 芯片 ────────────────────────┐
│  ┌──────────────┐  mailbox   ┌────────────────────────────────┐  │
│  │ GSP（RISC-V）│◄==========►│ host（macOS / NVRM.kext）       │  │
│  │  + Falcon    │ 0x110040/44│ 经 BAR0 MMIO 寄存器通信         │  │
│  │  （跑固件）   │ 0x110804/08└────────────────────────────────┘  │
│  └──────┬───────┘                                               │
│  ┌──────▼───────┐ ┌──────────┐ ┌───────────────────────────┐    │
│  │ 显示引擎       │ │ 拷贝引擎  │ │ GPU MMU（页表 / gpuva）     │    │
│  │ 0..3 共 4 head │ │ (CE)     │ │ 映射 VRAM / 系统内存        │    │
│  └──────┬───────┘ └──────────┘ └───────────────────────────┘    │
│  ┌──────▼──────────────────────────────┐                        │
│  │ VRAM（显存，如 8 GB）                 │                        │
│  └─────────────────────────────────────┘                        │
└──────────────────────────────────────────────────────────────────┘
```

| 部分 | 依据 |
|---|---|
| GSP = RISC-V + Falcon | `PGSP_FALCON_*`（0x110040/44/80/08/3c0）、`GSP_RISCV_CPUCTL`（0x111388）、`PGSP_MAILBOX(0/1)`、`PGSP_QUEUE_HEAD` |
| GSP 固件受保护内存 | `PFB_PRI_MMU_WPR2_ADDR_*`（0x1FA824/28） |
| 4 个显示 head | `NVRM_MAX_HEADS 4`；`nvfbheads` boot-arg |
| 拷贝引擎 | `nvrm_pageoff.c` 的 `ceutilsMemcopy` / `memdesc*` |
| GPU MMU / gpuva | `nvrm_gpuva.cpp` 的 `FERMI_VASPACE_A` + `NV50_MEMORY_VIRTUAL` + `MAP_MEMORY_DMA` |

### 3. 内存拓扑

```
        CPU 物理地址空间                          GPU 侧
 ┌───────────────────────────┐          ┌───────────────────────┐
 │ 系统 RAM                   │  DMA ───►│ GPU MMU               │
 │                           │          │  gpuva ──► VRAM       │
 │ BAR1 aperture（可放大到    │◄─窗口──  │        └► 系统 RAM     │
 │   4/8 GB，映进 VRAM）       │          │                       │
 │ BAR0（寄存器 MMIO，几 MB）  │          │ VRAM（显存）           │
 │ BAR3（另一个 aperture）     │          │                       │
 └───────────────────────────┘          └───────────────────────┘
```

- **BAR0** = 寄存器窗口（`NV_GPU_BAR_INDEX_REGS`）。
- **BAR1** = 显存大窗口（`NV_GPU_BAR_INDEX_FB`），`placeLargeBar1()` 摆位并可放大到 8 GB。
- **BAR3** = 另一个 64 位 aperture，与 BAR1 一起被搬。
- 显存大小由 `nvrm_fbinfo.cpp` 的 `FB_GET_INFO_V2` 查出，发布为 `VRAM,totalsize`。
- `nvrm_vram_abi.h` 的 `NVRM_VRAM_BAR1_BUDGET = 192 MB` 为经 BAR1 的显存分配预算。

### 4. 显示输出拓扑

```
GPU 显示引擎 ── head 0 ── connector ── 显示器
             ├─ head 1 ── connector ── 显示器
             ├─ head 2 ── connector
             └─ head 3 ── connector
        │
        └─ 扫描输出 → NVRMFB.kext（IOFramebuffer）+ NVRMAGDC.kext（显示策略）
```

- 每个 head 生成一个 `NVRMDisplay` nub，`NVRMFB.kext` 的 `NVRMNVDAFramebuffer` 匹配它。
- `fb/nvrm-fb.cpp` 有 `fHead` / `fConnector` / `fDisplay`、`flipToMemory`、scanout。
- `nvrm_vram_abi.h` 定义 `nvScanoutInfo`、`NVRMFlipRequest`（`nvFlipToSurface`）。

### 5. 硬件资源到驱动接口的映射

| 硬件资源 | 驱动如何使用 |
|---|---|
| BAR0 寄存器 | `IOMemoryDescriptor` 映射后读写（`os_map_kernel_space`） |
| BAR1/BAR3 显存窗口 | `fPCI->setDeviceMemory()`，供 NVK/accel 使用 |
| MSI 中断 | `IOFilterInterruptEventSource` → `rm_isr` |
| GSP 通信 | BAR0 mailbox 寄存器 + 共享内存 + 中断 |
| 显存/系统内存 DMA | GPU MMU（gpuva）+ host 侧 `IOMemoryDescriptor` |
| 显示扫描 | `IOFramebuffer` + flip notifier |
| GSP 固件文件 | `/Users/Shared/nvfw/nvidia/610.57.04/*.bin`（`nv_get_firmware`） |

### 6. 实测与覆盖范围

- **实测拓扑**：原生 PCIe Root Port + RTX 5060。
- **识别但不完整**：Thunderbolt eGPU（RTX 3070 失败证据 `kbusVerifyBar2 NV_ERR_MEMORY_ERROR`），BAR1 不搬。
- **不支持**：纯 USB-C（无 PCIe 隧道）、Apple Silicon、Pascal 及更早、Optimus 显示 mux。

一句话：

> CPU Root Complex → Root Port →（可选 Thunderbolt 桥链）→ NVIDIA GPU（内含 GSP RISC-V + 显示/拷贝引擎 + MMU + VRAM）→ 显示 connector。源码只对「Root Port 直连 GPU」链路做完整 BAR1 重排；Thunderbolt 链路会识别但退化为「用固件放好的 BAR1」。

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