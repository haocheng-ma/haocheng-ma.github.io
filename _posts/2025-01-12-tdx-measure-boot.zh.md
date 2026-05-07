---
layout: post
title: "Intel TDX: Measured Boot and Attestation in Grub Boot"
date: 2025-01-12
description: 如何在使用 grub 启动 TD guest 时构建可信链。
tags: security
lang: zh
hidden: true
permalink: /blog/2025/tdx-measure-boot/zh/
# giscus_comments: true
# related_posts: true
toc: true
---

Intel Trust Domain Extension（TDX）允许用户部署硬件隔离的虚拟机，称为 Trust Domain（TD）。TD 与虚拟机管理器（VMM）、hypervisor 以及宿主平台上其他非 TD 软件相互隔离。TD 的整个内存内容通过多密钥加密方式加密。Intel TDX 将宿主平台排除在 TD 的可信计算基（TCB）之外。

Intel TDX 的工作负载主要是租户希望在云环境中以 TD 形式运行的 VM 镜像。要在 TD 中运行 VM，需要由 CSP 提供的特殊 Open Virtual Machine Firmware（OVMF），例如 TDX Virtual Firmware（TDVF）。TDVF 是一个基于 EDK2 的项目，为基于 TDX 的虚拟机提供 UEFI 支持，用于启动 TD。TDVF 允许两种不同特性的配置，详见 [Configurations and Features](https://github.com/tianocore/edk2/blob/master/OvmfPkg/IntelTdx/README.md) 中的 [Appendix: TDVF Configurations](#jumpA)。

TD 启动过程中，对 TCB 的哈希链式度量会扩展到若干安全寄存器中（即度量启动）。这些寄存器的值会组合为一份报告，最终由证明密钥（attestation key）签名，生成一份 quote。在向工作负载提供数据之前，依赖方通过远程证明来核实软件确实运行在真实 Intel TDX 系统的 TD 内，并达到指定的安全级别。

_注：此处 TCB 指的是系统中提供安全环境的所有硬件、固件和软件组件，包括 CPU、SEAM 固件等硬件信息，以及 OVMF、bootloader（shim/grub）、内核等 guest 组件。_

**以上是 Intel TDX 的简要描述。在接下来的系列文章中，我们将聚焦于 TD 的度量启动与远程证明。**

## TD 启动类型

下图展示了 TD 启动类型与启动流程。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/td_guest_boot_process.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 1. TD Guest Boot Process
</div>

**Direct boot** 是指系统不经过中间引导加载器直接进入 OS 的启动过程。也可以称为带固件的直接内核启动：固件通过 Qemu 提供的 FwCfg 设备加载 kernel 和 initrd。

**Grub boot** 使用 Grub 引导加载器，提供更丰富的启动菜单选项，可选择不同的操作系统并自定义启动配置。也可称为 firmware-only boot：固件从磁盘镜像中启动一个 bootloader。

不同 TD 启动方式的详细流程见图 2。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/detailed_flow_for_different_td_boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. Detailed Flow for Different TD Boot
</div>

**今天我们深入讨论 Grub Boot 下的度量启动。**

在虚拟固件（OVMF）中，通过 `CoreLoadImageCommon()` 加载 EFI 镜像时会触发镜像处理函数 `DxeTpmMeasureBootHandler`。该处理函数将 FV、QEMU CFG、VMM Hob、Variable 等对象度量到度量寄存器（MR）中。随后在 bootloader ShimX64.efi 中，`TpmMeasureVariable()` 将 Secure Boot 证书度量到 MR 中；如果未启用 Secure Boot，VM 镜像中可能不包含 ShimX64.efi。最终，bootloader GrubX64.efi 将 kernel 二进制及其命令行、initrd 二进制以及 grub 的模块度量到 MR 中。

上述过程借鉴了 TCG 可信启动链。不同之处在于：它不信任 hypervisor，并避免使用可变的非易失性存储（否则会导致 MR 变化）；而且其信任链可追溯至支持 TDX 的硬件。

## UEFI 启动时序

UEFI 允许通过加载 UEFI 驱动和 UEFI 应用镜像来扩展平台固件。UEFI 驱动与应用加载后可以访问所有 UEFI 定义的运行时服务和启动服务。启动时序见下图 3。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/uefi_booting_sequence.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 3. UEFI Booting Sequence
</div>

UEFI 可将 OS loader 和平台固件的启动菜单整合为一个统一的平台固件菜单。这些菜单允许从任意启动介质的任意分区选择任意 UEFI OS loader，只要 UEFI 启动服务支持该介质。

OS loader 作为 UEFI 应用运行，通过 Boot Services（BS）和 Runtime Services（RT）调用 UEFI 服务。随后调用 `ExitBootServices()` 终止 BS 并释放其资源，之后 OS 仅能使用 RT。

UEFI 启动阶段的详细介绍见 [UEFI boot phases](https://secret.club/2020/05/26/introduction-to-uefi-part-1.html#uefi-boot-phases) 中的 [Appendix: UEFI Boot Phase](#jumpB)。

## 度量启动组件

如图 4，内核之前的 pre-boot 环境包括 TDVF/OVMF 阶段以及 bootloader 阶段（shim 和 grub）。整条启动链通过 `EFI_CC_MEASUREMENT_PROTOCOL` 被度量进 Runtime Measurement Register（RTMR）。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/td_measurement_process.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 4. TD Measurement Process
</div>

类似于 TCG event log，`EFI_CC_MEASUREMENT_PROTOCOL` 将事件记入 confidential computing event log（CCEL）ACPI 表，并将度量哈希扩展到对应的 RTMR 寄存器。CCEL 表中的事件日志可在 TD guest 内回放，以校验 RTMR 的值。

EDK2 中的度量启动组件如下。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/measured_boot_component_in_edk2.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 5. Measured Boot Component in EDK2
</div>

[`SecTdxHelperLib`](https://github.com/tianocore/edk2/tree/master/OvmfPkg/IntelTdx/TdxHelperLib) 在 SEC 阶段提供度量函数。
[`EFI_CC_MEASUREMENT_PROTOCOL`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Protocol/CcMeasurement.h) 协议抽象了 UEFI guest 环境中的机密计算（CC）度量操作。
[TdTcg2Dxe.c](https://github.com/tianocore/edk2/blob/master/OvmfPkg/Tcg/TdTcg2Dxe/TdTcg2Dxe.c) DXE 驱动处理 DXE 阶段的度量。
[`DxeTpm2MeasureBootLib`](https://github.com/tianocore/edk2/tree/master/SecurityPkg/Library/DxeTpm2MeasureBootLib) 处理 PE 镜像度量和 GPT 度量。所有事件类型的定义见 [`UefiTcgPlatform.h`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h)。

下面根据 [Confidential Computing in UEFI Specification](https://uefi.org/specs/UEFI/2.10/38_Confidential_Computing.html#) 简述 `EFI_CC_MEASUREMENT_PROTOCOL`。

如果具备 CC 能力的虚拟固件支持度量，则它应提供使用新 GUID `EFI_CC_MEASUREMENT_PROTOCOL_GUID` 的 `EFI_CC_MEASUREMENT_PROTOCOL`，用以上报事件日志并提供哈希能力。

```c
#define EFI_CC_MEASUREMENT_PROTOCOL_GUID  \
  { 0x96751a3d, 0x72f4, 0x41a6, { 0xa7, 0x94, 0xed, 0x5d, 0x0e, 0x67, 0xae, 0x6b }}
extern EFI_GUID  gEfiCcMeasurementProtocolGuid;

# Protocol Interface Structure
typedef struct _EFI_CC_MEASUREMENT_PROTOCOL {
  EFI_CC_GET_CAPABILITY          GetCapability;
  EFI_CC_GET_EVENT_LOG           GetEventLog;
  EFI_CC_HASH_LOG_EXTEND_EVENT   HashLogExtendEvent;
  EFI_CC_MAP_PCR_TO_MR_INDEX     MapPcrToMrIndex;
} EFI_CC_MEASUREMENT_PROTOCOL;
```

该协议接口包含四个成员：

- GetCapability：提供协议能力信息与状态信息。
- GetEventLog：允许调用方获取指定事件日志的地址及其末尾条目。
- HashLogExtendEvent：允许调用方扩展并可选地记录事件，无需了解实际的 CC 命令。
- MapPcrToMrIndex：向调用方提供 TPM PCR 到 CC MR 的映射信息。

**Intel TDX 的映射关系**

下表展示了 TPM Platform Configuration Register（PCR）索引映射以及 Intel TDX 下 CC event log MR 索引的含义。MRTD 指 Trust Domain Measurement Register，RTMR 指 Runtime Measurement Register。TD MR 数量少于 TPM PCR，典型映射如下：

| TPM PCR Index | Typical Usage of Measurement Registers                | TDX MR Index |
| ------------- | ----------------------------------------------------- | ------------ |
| 0             | FirmwareCode (BFV, including init page table)         | MRTD         |
| 1             | FirmwareData (CFV, TD Hob, ACPI Table, Boot Variable) | RTMR[0]      |
| 2             | Option ROM code                                       | RTMR[1]      |
| 3             | Option ROM code                                       | RTMR[1]      |
| 4             | OS loader code                                        | RTMR[1]      |
| 5             | GUID partition table (GPT)                            | RTMR[1]      |
| 6             | N/A                                                   | N/A          |
| 7             | Secure Boot Configuration                             | RTMR[0]      |
| 8~15          | TD OS measurement                                     | RTMR[2]      |

_注：RTMR[3] 保留用于特殊用途，例如 virtual TPM。如果这些专用场景不需要 RTMR[3]，用户可以灵活使用。_

MRTD 与 RTMR 的典型用途如下，更多细节见 [8.1 Measurement Register Usage in TD](https://cdrdv2.intel.com/v1/dl/getContent/733585)：

- MRTD 用于 TDVF 代码（对应 PCR[0]）。
- RTMR[0] 用于 TDVF 配置（对应 PCR[1,7]），应遵循 TCG Platform Firmware Profile（PFP）规范。
- RTMR[1] 用于 TDVF 加载的组件，例如 OS loader（对应 PCR[4,5]），应遵循 TCG PFP 规范。
- RTMR[2] 用于 OS 组件，例如 OS kernel、initrd 与应用程序（对应 PCR[8~15]），使用方式由 OS 决定。
- RTMR[3] 仅保留给特殊用途。

## 度量启动流程

总体而言，事件在启动时序图中的过渡点（虚线处）被度量。由此建立一条信任链：虚拟固件 → bootloader → OS → 应用。

**度量 TdHob 与 Configuration FV（Cfv）**

[4.2 TD Hand-Off Block (HOB)](https://cdrdv2.intel.com/v1/dl/getContent/733585) 和 [3.2 Configuration Firmware Volume (CFV)](https://cdrdv2.intel.com/v1/dl/getContent/733585) 是 Host VMM 提供的外部数据，TD guest 默认不信任。因此必须对其进行校验、度量并扩展到 TD 的 RTMR 寄存器。与此同时会创建两个 `EFI_CC_EVENT_HOB`，这两个 GUID HOB 分别携带 TdHobList 和 Configuration FV 的哈希值。在 DXE 阶段，可基于这两个 GUID HOB 创建 `EFI_CC_EVENT`。

Configuration Firmware Volume 包含所有预置数据，该区域只读。一种可能用途是存放 UEFI Secure Boot 变量内容，例如 PK、KEK、db、dbx。

TD HOB list 用于从 VMM 向 TDVF 传递信息。TD HOB 必须包含 PHIT HOB 和 Resource Descriptor HOB，其他 HOB 可选。

- TD HOB 的第一个 HOB 必须是 PHIT HOB。EfiMemoryTop、EfiMemoryBottom、EfiFreeMemoryTop、EfiFreeMemoryBottom 都应为 0。
- TD HOB 必须至少包含一个 Resource Description HOB，用以声明物理内存资源。

**_0._** `SecTdxHelperLib` 是 TdxHelperLib 的 SEC 实例。它在 SEC 阶段为 TDX 提供以下功能：

- `TdxHelperMeasureTdHob` 度量/扩展 TdHob，并把度量值存入 workarea。
- `TdxHelperMeasureCfvImage` 度量/扩展 Configuration FV 镜像，并把度量值存入 workarea。
- `TdxHelperBuildGuidHobForTdxMeasurement` 为 TDX 度量构建 GuidHob。

**度量 Secure Boot Policy 到 PCR[7]/RTMR[0]**

_1._ UEFI Debug Mode。

如果平台提供固件调试模式，则必须以 `EV_EFI_ACTION` 度量字符串 "UEFI Debug Mode"。该逻辑在 TdTcg2Dxe.c 的 `MeasureSecureBootPolicy()` 中实现，依据 [`PcdFirmwareDebuggerInitialized`](https://github.com/tianocore/edk2/blob/master/SecurityPkg/SecurityPkg.dec) 判定。

_2._ `SecureBoot` 变量的内容。

_3._ `PK` 变量的内容。

_4._ `KEK` 变量的内容。

_5._ `EFI_IMAGE_SECURITY_DATABASE` 变量（DB）的内容。

_6._ `EFI_IMAGE_SECURITY_DATABASE1` 变量（DBX）的内容。

_7._ 如存在且非空，则度量 `EFI_IMAGE_SECURITY_DATABASE2`（DBT）变量的内容。

UEFI Secure Boot 相关变量——"SecureBoot"、"PK"、"KEK"、"db"、"dbx"——由 TdTcg2Dxe.c 的 `ReadAndMeasureSecureVariable()` 无条件度量，事件类型为 [`EV_EFI_VARIABLE_DRIVER_CONFIG`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h)。若变量不存在，则度量一条零长度的 UEFI 变量条目。"dbt" 与 "dbr" 变量在存在时才被 `MeasureAllSecureVariables()` 度量。

_8._ Separator。

PCR7 的 [`EV_SEPARATOR`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h) 在 UEFI 变量就绪后由 TdTcg2Dxe.c 的 `MeasureSecureBootPolicy()` 处理，位于 `MeasureAllSecureVariables()` 之后、`ReadyToBoot` 事件信号之前。原因是 PCR7 的 `EV_SEPARATOR` 必须位于 SecureBootPolicy（Configure）与 ImageVerification（Authority）之间。

**度量 Boot Variable 到 PCR[1]/RTMR[0]**

_9._ UEFI BootOrder 变量和 Boot#### 变量（仅 device path）。

UEFI 启动相关变量（例如 "BootOrder" 和 "Boot####"）由 TdTcg2Dxe.c 的 `ReadAndMeasureBootVariable()` 度量，事件类型为 [`EV_EFI_VARIABLE_BOOT`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h)。`MeasureAllBootVariables()` 在变量存在时将其度量。

**选择启动设备后，**

_10._ 启动动作 "Calling EFI Application from Boot Option"，表示 Boot Manager 正在尝试从某个启动选项执行代码（**PCR[4]/RTMR[1]**）。

该启动动作由 TdTcg2Dxe.c 的 `OnReadyToBoot()` 度量。调用启动选项之前，度量 "Calling EFI Application from Boot Option"；启动选项返回之后，度量 "Returning from EFI Application from Boot Option"。

_11._ Separator，用以分隔离开 pre-boot 环境与进入 post-boot 环境（**PCR[0~6]/RTMR[1]**）。

_12._ **[可选] 若启用 UEFI Secure Boot**，度量 `EFI_IMAGE_SECURITY_DATABASE` 中用于校验 UEFI 镜像的那一条条目（**PCR[7]/RTMR[0]**）。

启用 UEFI Secure Boot 时，[`DxeImageVerificationLib`](https://github.com/tianocore/edk2/tree/master/SecurityPkg/Library/DxeImageVerificationLib) 依据镜像签名数据库中 [`EFI_SIGNATURE_LIST`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Guid/ImageAuthentication.h) 里的 [`EFI_SIGNATURE_DATA`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Guid/ImageAuthentication.h) 来校验 PE 镜像签名。若用某个 `EFI_SIGNATURE_DATA` 校验了镜像，则 [Measurement.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/Measurement.c) 的 `MeasureVariable()` 会以 [`EV_EFI_VARIABLE_AUTHORITY`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h) 度量该 `EFI_SIGNATURE_DATA`。

_13._ GUID Partition Table（GPT）磁盘布局（**PCR[5]/RTMR[1]**）。

当系统从磁盘中某个 GUID 命名分区启动时，需要度量 GPT 磁盘布局。由 [DxeTpm2MeasureBootLib.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeTpm2MeasureBootLib/DxeTpm2MeasureBootLib.c) 的 `Tcg2MeasureGptTable()` 在 `DxeTpm2MeasureBootHandler()` 中完成。

_14._ 选中的 UEFI 应用代码 PE/COFF 镜像，即 OS loader（**PCR[4]/RTMR[1]**）。

第三方 UEFI 应用（例如 UEFI shell 工具、标准 OS loader 或 OEM 启动选项）由 DxeTpm2MeasureBootLib.c 的 `Tcg2MeasurePeImage()` 在 `DxeTpm2MeasureBootHandler()` 中度量，事件类型为 [`EV_EFI_BOOT_SERVICES_APPLICATION`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h)。若某 UEFI 应用是 DXE 阶段分派的 FV，则无论该 FV 本身是否被度量，都会度量到 PCR4。

**随后 OS loader —— [Grub2](https://git.savannah.gnu.org/cgit/grub.git/) —— 将可信启动链从虚拟固件延伸到 OS。**

_注：此处不讨论 [Shim](https://github.com/rhboot/shim) 组件，因为我们只关注度量启动。Shim 用于将 UEFI Secure Boot 概念扩展到 Linux。_

_15._ Grub2 度量配置文件（例如 grub.cfg）、grub 命令、kernel 二进制、kernel 命令行和 initrd 二进制（**PCR[8, 9]/RTMR[2]**）。PCR[8] 对应命令行字符串，PCR[9] 对应文件二进制，如下表所示。

为支持机密计算平台上的度量，两个补丁已合入上游：

- [efi/tpm: Add EFI_CC_MEASUREMENT_PROTOCOL support](https://git.savannah.gnu.org/cgit/grub.git/commit/?id=4c76565b6cb885b7e144dc27f3612066844e2d19)
- [commands/efi/tpm: Re-enable measurements on confidential computing platforms](https://git.savannah.gnu.org/cgit/grub.git/commit/?id=86df79275d065d87f4de5c97e456973e8b4a649c)

| PCR Index | PCR Usage                                                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 8         | Grub command line: All executed commands (including those from configuration files) will be logged and measured as entered with a prefix of "grub cmd:" |
|           | Kernel command line: Any command line passed to a kernel will be logged and measured as entered with a prefix of "kernel cmdline:"                      |
|           | Module command line: Any command line passed to a kernel module will be logged and measured as entered with a prefix of "module cmdline:"               |
| 9         | Files: Any file read by GRUB will be logged and measured with a descriptive text corresponding to the filename.                                         |

[tpm.c](https://github.com/rhboot/grub2/blob/master/grub-core/commands/tpm.c) 将 `grub_tpm_verify_string()` 和 `grub_tpm_verify_write()` 注册到 grub_file_verifier 结构体。它们由 [verifiers.c](https://github.com/rhboot/grub2/blob/master/grub-core/commands/verifiers.c) 中的 `grub_verify_string()` 和 `grub_verifiers_open()` 调用。

当 grub2 执行诸如 `GRUB_VERIFY_MODULE_CMDLINE`、`GRUB_VERIFY_KERNEL_CMDLINE`、`GRUB_VERIFY_COMMAND` 等命令行，或 [cmdline.c](https://github.com/rhboot/grub2/blob/master/grub-core/lib/cmdline.c) 中的 `grub_create_loader_cmdline()` 时，会调用 `grub_verify_string()`。最终 `grub_tpm_verify_string()` 调用 `grub_tpm_measure`，然后由 `grub_cc_log_event` 将字符串度量到 **PCR[8]/RTMR[2]**。

`grub_verifiers_open()` 注册为 [`file.h`](https://github.com/rhboot/grub2/blob/master/include/grub/file.h) 中的 grub_file_filter 之一。每当 grub 调用 [file.c](https://github.com/rhboot/grub2/blob/master/grub-core/kern/file.c) 的 `grub_file_open()` 时，都会触发该过滤器。最终 `grub_tpm_verify_write()` 调用 `grub_tpm_measure`，然后由 `grub_cc_log_event` 将文件二进制度量到 **PCR[9]/RTMR[2]**。

_16._ 启动动作 "Exit Boot Services Invocation"，表示 Boot Manager 已向 UEFI 发出结束 Boot Services 的调用（**PCR[5]/RTMR[1]**）。

_17._ 启动动作 "Exit Boot Services Returned with Success"，表示 UEFI 成功退出 Boot Services，pre-OS 环境已终止（**PCR[5]/RTMR[1]**）。

ExitBootServices 动作由 TdTcg2Dxe.c 度量。成功时调用 `OnExitBootServices()`，失败时调用 `OnExitBootServicesFailed()`。

**若 Security Boot Policy 在首次度量之后、`ExitBootServices()` 完成之前发生变更，**

_18._ 平台必须重启，或把这些变量重新度量到 **PCR[7]/RTMR[0]**。另外，对任何 Secure Boot 变量的常规更新都应在 PCR[7] 的首次度量之前，或 `ExitBootServices()` 调用完成之后进行。

UEFI Secure Boot 变量的更新在 Variable [`RuntimeDxe`](https://github.com/tianocore/edk2/tree/master/MdeModulePkg/Universal/Variable/RuntimeDxe) 中度量。如果上述任一 Secure Boot 相关变量被更新，[Measurement.c](https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Universal/Variable/RuntimeDxe/Measurement.c) 的 `MeasureVariable()` 会以 `EV_EFI_VARIABLE_DRIVER_CONFIG` 度量新的数据。

## 远程证明

我们从验证方视角分析如何校验 TD Quote，不涉及 Quote 生成或与证明方的交互。总体可分为：Quote 校验、事件日志回放、事件解析。

### Quote 校验

度量启动根信任（RoT）可以发布一份 MR 报告，通常称为 quote 或 attestation report。该 quote 通常是对 MR 摘要的数字签名。

验证方随后确认 MR 摘要是由可信密钥——RoT for Reporting（RTR）——签发。此 RTR 即证明密钥（attestation key），通常是一把经过认证的密钥，用于对 MR 报告签名。这种认证在 IETF RATS 架构中称为 Endorsement。更多 quote 校验的细节见 [Intel SGX-based Data Center Attestation Primitives (DCAP)](https://github.com/intel/SGXDataCenterAttestationPrimitives)。

### 事件日志回放

事件日志回放是把原始事件日志反序列化后，利用其中的事件重新计算所有度量寄存器的值。每个事件包含一个摘要和度量寄存器索引。验证方创建一组模拟度量寄存器，对每个事件将其摘要扩展到对应的模拟寄存器中。最后将模拟寄存器的值与第一步中 quote 上报的真实度量寄存器值比对。

Intel 提供了一个 Python 库 [tdx-tools](https://github.com/canonical/tdx/tree/main/tests/lib/tdx-tools/src/tdxtools) 用于校验 TD 度量。其流程分为三步，完整步骤可参考 tdx-tools 中定义的 `VerifyActor` 类。

- 从 CCEL ACPI 表获取 TD event log。
- 通过 Linux attestation driver 获取 TDREPORT。
- 对比 TDREPORT 中的 RTMR 值与事件日志回放得到的 RTMR 值。

两者应当完全一致，表示被度量的内容未被篡改。

```python
class VerifyActor:
    """
    Actor to verify the RTMR
    """

    def _verify_single_rtmr(self, rtmr_index: int, rtmr_value_1: RTMR,
        rtmr_value_2: RTMR) -> None:

        if rtmr_value_1 == rtmr_value_2:
            LOG.info("RTMR[%d] passed the verification.", rtmr_index)
        else:
            LOG.error("RTMR[%d] did not pass the verification", rtmr_index)

    def verify_rtmr(self) -> None:
        """
        Get TD report and RTMR replayed by event log to do verification.
        """
        # 1. Read CCEL from ACPI table at /sys/firmware/acpi/tables/CCEL
        ccelobj = CCEL.create_from_acpi_file()
        if ccelobj is None:
            return

        # 2. Get the start address and length for event log area
        td_event_log_actor = TDEventLogActor(
            ccelobj.log_area_start_address,
            ccelobj.log_area_minimum_length)

        # 3. Collect event log and replay the RTMR value according to event log
        td_event_log_actor.replay()

        # 4. Read TD REPORT via TDCALL.GET_TDREPORT
        td_report = TdReport.get_td_report()

        # 5. Verify individual RTMR value from TDREPORT and recalculated from
        #    event log
        self._verify_single_rtmr(
            0,
            td_event_log_actor.get_rtmr_by_index(0),
            RTMR(bytearray(td_report.td_info.rtmr_0)))

        self._verify_single_rtmr(
            1,
            td_event_log_actor.get_rtmr_by_index(1),
            RTMR(bytearray(td_report.td_info.rtmr_1)))

        self._verify_single_rtmr(
            2,
            td_event_log_actor.get_rtmr_by_index(2),
            RTMR(bytearray(td_report.td_info.rtmr_2)))

        self._verify_single_rtmr(
            3,
            td_event_log_actor.get_rtmr_by_index(3),
            RTMR(bytearray(td_report.td_info.rtmr_3)))
```

### 事件解析

事件解析是从事件中提取信息的过程。验证方据此对照评估策略（appraisal policy）、endorsements 和 reference values 做出判断。

在 [go-eventlog](https://github.com/google/go-eventlog) 中，`ReplayAndExtract` 解析一条 CC event log，并依据哈希算法在指定的 RTMR bank 上回放事件日志（即第二步）。随后从已校验的日志中抽取事件信息写入 FirmwareLogState。该 FirmwareLogState 包含固件、Secure Boot 配置、bootloader（例如 grub）以及 kernel（含命令行和 initramfs）的详细信息。

```go
// FirmwareLogState extracts event info from a verified TCG PC Client event
// log into a FirmwareLogState.
//
// It is the caller's responsibility to ensure that the passed events have
// been replayed (e.g., using `tcg.ParseAndReplay`) against a verified measurement
// register bank.
func FirmwareLogState(events []tcg.Event, hash crypto.Hash, registerCfg registerConfig, opts Opts) (*pb.FirmwareLogState, error) {
	var joined error
	tcgHash, err := tpm2.HashToAlgorithm(hash)
	if err != nil {
		return nil, err
	}

	platform, err := registerCfg.PlatformExtracter(hash, events)
	if err != nil {
		joined = errors.Join(joined, err)
	}
	sbState, err := SecureBootState(events, registerCfg)
	if err != nil {
		joined = errors.Join(joined, err)
	}
	efiState, err := EfiState(hash, events, registerCfg)

	if err != nil {
		joined = errors.Join(joined, err)
	}

	var grub *pb.GrubState
	var kernel *pb.LinuxKernelState
	if opts.Loader == GRUB {
		grub, err = registerCfg.GRUBExtracter(hash, events)

		if err != nil {
			joined = errors.Join(joined, err)
		}
		kernel, err = LinuxKernelStateFromGRUB(grub)
		if err != nil {
			joined = errors.Join(joined, err)
		}
	}
	return &pb.FirmwareLogState{
		Platform:    platform,
		SecureBoot:  sbState,
		Efi:         efiState,
		RawEvents:   tcg.ConvertToPbEvents(hash, events),
		Hash:        pb.HashAlgo(tcgHash),
		Grub:        grub,
		LinuxKernel: kernel,
	}, joined
}
```

### 参考度量值

最后还有一个相当棘手的问题：如何为 FirmwareLogState 生成参考度量值。目前 tdx 仓库中该问题仍未关闭，见 [Calculate measurements outside of TDX #263](https://github.com/canonical/tdx/issues/263)。受 cc-api 仓库中 [cvm-image-rewriter](https://github.com/cc-api/cvm-image-rewriter) 工具的启发，我们可以设计一个插件来计算 VM 镜像的参考度量值。

cvm-image-rewriter 基于插件架构，用于定制机密计算 VM guest，包括 guest 镜像、配置、OVMF 固件等。现有插件如下表所示。

| Name                             | Descriptions                                            |
| -------------------------------- | ------------------------------------------------------- |
| 01-resize-image                  | Resize the input qcow2 image                            |
| 02-motd-welcome                  | Customize the login welcome message                     |
| 03-netplan                       | Customize the netplan.yaml                              |
| 04-user-authkey                  | Add auth key for user login instead of password         |
| 05-readonly-data                 | Fix some file permission to ready-only                  |
| 06-install-tdx-guest-kernel      | Install MVP TDX guest kernel                            |
| 07-device-permission             | Fix the permission for device node                      |
| 08-ccnp-uds-directory-permission | Fix the permission for CCNP UDS directory               |
| 60-initrd-update                 | Update the initrd image                                 |
| 97-sample                        | plugin customization example                            |
| 98-ima-enable-simple             | Enable IMA (Integrity Measurement Architecture) feature |

定制完 VM 镜像后，一个插件（可命名为 reference-measurement-calculator）可以读取 grub 和 kernel，计算镜像与配置的摘要。由此得到 VM 镜像的参考度量值，用于最终校验。

接下来是如何获取固件的参考度量值。根据 TDVF Binary Layout，可以解析虚拟固件，定位 BFV 和 CFV，然后计算它们的参考度量值。这里推荐参考 contrast 仓库中的工具 [tdx-measure](https://github.com/edgelesssys/contrast/blob/main/tools/tdx-measure/tdvf/tdvf.go)。

该工具还尝试了另一种证明方式：对给定的固件和 kernel 文件进行 RTMR 预计算（见 `newRtMrCmd()` 函数），再将 TDREPORT 中的 RTMR 与预计算 RTMR 比对。这种方式省去了事件日志，从而提升校验效率。需要指出，这种方式仅适用于相对固定的场景。在大多数场景中，事件日志仍不可或缺，因为固件、bootloader、OS 和应用都可以按任意顺序追加任意事件的度量。

```go
func newRtMrCmd() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "rtmr -f OVMF.fd -k bzImage [0|1|2|3]",
		Short: "calculate the RTMR for a firmware and kernel file",
		Long: `Calculate the RTMR for a firmware and kernel file.

		This will parse the firmware according to the TDX Virtual Firmware Design Guide
		and/or hash the kernel and pre-calculate a given RTMR.`,
		Args:      cobra.MatchAll(cobra.ExactArgs(1), cobra.OnlyValidArgs),
		ValidArgs: []string{"0", "1", "2", "3"},
		RunE:      runRtMr,
	}
	cmd.Flags().StringP("firmware", "f", "OVMF.fd", "path to firmware file")
	if err := cmd.MarkFlagFilename("firmware", "fd"); err != nil {
		panic(err)
	}
	cmd.Flags().StringP("kernel", "k", "bzImage", "path to kernel file")
	if err := cmd.MarkFlagFilename("kernel"); err != nil {
		panic(err)
	}
	cmd.Flags().StringP("initrd", "i", "initrd.zst", "path to initrd file")
	if err := cmd.MarkFlagFilename("initrd"); err != nil {
		panic(err)
	}
	cmd.Flags().StringP("cmdline", "c", "", "kernel command line")
	return cmd
}
```

## 附录

<span id="jumpA"></span>

### TDVF Configurations

**Config-A**：

- 将*基础* TDVF 特性合入现有的 `OvmfX64Pkg.dsc`。（与既有 SEV 对齐）
- 威胁模型：VMM 不排除在 TCB 之外。（不改变现状）
- `OvmfX64Pkg.dsc` 包含 SEV/TDX/normal OVMF 的基本启动能力，最终二进制可在 SEV/TDX/normal OVMF 上运行。
- 不修改 OvmfPkgX64 现有镜像布局。
- 无需移除现有特性。
- TD 与 Non-TD 下均不跳过 PEI 阶段。
- 支持基于 RTMR 的度量。
- 度量来自 Host VMM 的外部输入，例如 TdHob、CFV。
- 度量其他外部输入，例如 FW_CFG 数据、os loader、initrd 等。

**构建 TDVF（Config-A）目标：**

[OvmfPkg: Support Tdx measurement in OvmfPkgX64](https://git.codelinaro.org/linaro/dcap/edk2/-/commit/4d37059d8e1eeda124270a158416795605327cbd)

```shell
cd /path/to/edk2
source edksetup.sh
build.sh -p OvmfPkg/OvmfPkgX64.dsc -a X64 -t GCC5
```

**Config-B**：

- 在 TDX 专用目录中新增独立的 `IntelTdx.dsc`，提供*完整*特性的 TDVF。（与既有 SEV 对齐）
- 威胁模型：VMM 排除在 TCB 之外。（需要必要改动以防御来自 VMM 的攻击）
- `IntelTdx.dsc` 包含 TDX/normal OVMF 的基本启动能力，最终二进制可在 TDX/normal OVMF 上运行。
- 将来可能与 AmdSev.dsc 合并，但目前不合并；具体时间未知。只有 Intel 与 AMD 双方都认为各自方案足够成熟时，才会在社区中协调合并。
- 必须新增必要的安全特性作为强制要求，例如基于 RTMR 的可信启动支持。
- 必须度量来自 Host VMM 的外部输入，例如 TdHob、CFV。
- 必须度量其他外部输入，例如 FW_CFG 数据、os loader、initrd 等。
- 必须移除不必要的攻击面，例如网络协议栈。

**构建 TDVF（Config-B）目标：**

[OvmfPkg: Introduce IntelTdxX64 for TDVF Config-B](https://git.codelinaro.org/linaro/dcap/edk2/-/commit/44a53a3bdd9c76e37f1750b5aa6a745de5d77391)

```shell
cd /path/to/edk2
set PACKAGES_PATH=/path/to/edk2/OvmfPkg
source edksetup.sh
build.sh -p OvmfPkg/IntelTdx/IntelTdxX64.dsc -a X64 -t GCC5
```

<span id="jumpB"></span>

### UEFI Boot Phase

UEFI 有六个主要启动阶段，每一阶段都是平台初始化过程中的关键环节。这些阶段合称 Platform Initialization（PI）。

_1._ Security（SEC）

UEFI 启动的第一个阶段，通常用于：初始化临时内存、充当系统信任根，并向 Pre-EFI 核心阶段提供信息。该信任根确保 PI 中执行的任何代码都经过密码学校验（数字签名），从而建立 "secure boot" 环境。

_2._ Pre-EFI Initialization（PEI）

启动过程的第二阶段，仅使用 CPU 当前可用的资源调度 Pre-EFI Initialization Modules（PEIM）。PEIM 用于完成启动关键操作的初始化（例如内存初始化），并将控制权交给 Driver Execution Environment（DXE）。

_3._ Driver Execution Environment（DXE）

DXE 阶段完成大部分系统初始化。DXE 运行所需的内存在 PEI 阶段分配并初始化；控制权交给 DXE 后，DXE Dispatcher 被触发，负责加载并执行硬件驱动、运行时服务以及 OS 启动所需的各类启动服务。

_4._ Boot Device Selection（BDS）

DXE Dispatcher 执行完所有 DXE 驱动后，控制权交给 BDS。BDS 负责初始化控制台设备及其他尚未初始化的设备；随后加载并执行所选启动项（OS loader），为 Transient System Load（TSL）做准备。

_5._ Transient System Load（TSL）

此阶段中，PI 流程位于启动项选择与交付给主操作系统之间。可以调用如 UEFI shell 之类的应用，或（更常见地）运行 bootloader 以准备最终 OS 环境。通常由 bootloader 通过 `ExitBootServices()` 终止 UEFI Boot Services；但 OS 本身也可以做此动作，例如启用 `CONFIG_EFI_STUB` 的 Linux kernel。

_6._ Runtime（RT）

最后一个阶段，控制权正式交给 OS。兼容 UEFI 的 OS 接管系统。UEFI Runtime Services 仍对 OS 可用，例如用于从 NVRAM 查询和写入变量。
