---
layout: post
title: "Intel TDX: Measured Boot and Remote Attestation with GRUB"
date: 2025-01-12
description: GRUB 启动的 TD 如何从固件一路度量到内核，以及验证方如何校验这条启动链。
tags: security
lang: zh
hidden: true
permalink: /blog/2025/tdx-measure-boot/zh/
giscus_comments: true
# related_posts: true
toc: true
---

Intel Trust Domain Extensions（TDX）把虚拟机放入由硬件隔离的 Trust Domain（TD）中，使 TD 的内存和执行状态不直接暴露给虚拟机管理器（VMM）、hypervisor 及宿主机上的其他非 TD 软件。云服务提供商仍负责启动和调度虚拟机，但不再因掌控宿主软件栈而自动进入 TD 的可信计算基（Trusted Computing Base，TCB）。

隔离只能回答“宿主是否能直接读取或修改 TD”，不能回答“TD 启动了什么”。后一个问题由度量启动（measured boot）和远程证明（remote attestation）共同回答：

1. 固件、启动配置、GRUB、内核和 initrd 等对象在执行或加载时被哈希；
2. 每个摘要按顺序扩展（extend）进 Trust Domain Measurement Register（MRTD）或 Runtime Measurement Register（RTMR），同时写入事件日志；
3. TDX Quote 对包括这些度量值在内的 TD 状态提供可验证的密码学证据；
4. 验证方先校验 Quote，再回放事件日志，并按自己的参考值和策略判断这次启动是否可接受。

TD 内的 UEFI 固件通常由 TDX Virtual Firmware（TDVF）提供；它基于 EDK2/OVMF，负责建立 TD 的固件执行环境并加载后续组件。本文是一篇机制说明，聚焦采用 GRUB 的 TD 启动路径：先说明 TDVF 如何把度量链交给 GRUB，再说明验证方如何把 Quote、事件日志和参考值连接起来。文中的 TCB 取决于验证策略；CPU 和 TDX Module/SEAM 固件属于平台侧基础，TDVF、GRUB、内核等 guest 组件是否被信任，则取决于验证方接受的参考值。TDVF 的两种配置及其不同威胁模型见[附录](#jumpA)。

## TD 启动类型

先从两条常见启动路径看本文的范围。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/td_guest_boot_process.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 1. TD Guest Boot Process
</div>

**Direct boot** 不经过 GRUB 等中间引导加载器。固件通过 QEMU 提供的 `fw_cfg` 接口取得 kernel、initrd 和命令行，然后直接启动内核。

**GRUB boot** 则由固件从磁盘镜像加载 UEFI bootloader，再由 GRUB 读取配置、选择内核并加载 initrd。这条路径多了一层软件和配置，也因此需要更长的度量链。

图 2 展开了两条路径。本文只讨论右侧的 GRUB boot。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/detailed_flow_for_different_td_boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. Detailed Flow for Different TD Boot
</div>

从机制上看，这条链有三个交接点：TDVF 度量自身配置和外部输入；UEFI 镜像加载路径度量 shim/GRUB 等 PE/COFF 镜像；GRUB 再度量配置、命令、kernel 和 initrd。若启用 UEFI Secure Boot，shim 还可以把发行版信任链延伸到 GRUB 和内核；但 Secure Boot 负责阻止未获授权的代码执行，度量启动负责记录实际加载了什么，两者不能互相替代。

这套事件类型和 PCR 语义沿用了 TCG 可信启动模型，TDX 则把事件映射到 MRTD/RTMR，并把硬件与 TDX Module 纳入 Quote 所证明的平台状态。TDVF 配置仍然决定 VMM 是否真正排除在 TCB 之外，不能仅凭“使用了 TDX”作出判断。

## UEFI 启动时序

GRUB 在这条路径中以 UEFI 应用的身份运行。理解它与固件的边界，需要先看图 3 所示的 UEFI 启动时序。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/uefi_booting_sequence.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 3. UEFI Booting Sequence
</div>

固件通过 Boot Services（BS）加载并启动所选的 OS loader；GRUB 等 loader 也以 UEFI 应用的身份调用这些服务。内核准备就绪后，loader 调用 `ExitBootServices()` 结束 BS 并释放相关资源，此后 OS 只能继续使用 UEFI Runtime Services（RT）。因此，`ExitBootServices()` 既是执行边界，也是事件日志中区分 pre-boot 与 post-boot 的关键节点。

UEFI 启动阶段的详细介绍见 [UEFI boot phases](https://secret.club/2020/05/26/introduction-to-uefi-part-1.html#uefi-boot-phases) 中的 [Appendix: UEFI Boot Phase](#jumpB)。

## 度量启动组件

如图 4，内核之前的 pre-boot 环境包括 TDVF/OVMF 和 bootloader（可选的 shim 以及 GRUB）两个阶段。UEFI 组件通过 `EFI_CC_MEASUREMENT_PROTOCOL` 记录事件，并把摘要扩展进 Runtime Measurement Register（RTMR）；MRTD 则记录 TD 初始构建阶段的度量。

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/td_measurement_process.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 4. TD Measurement Process
</div>

`EFI_CC_MEASUREMENT_PROTOCOL` 沿用 TCG event log 的基本模型：事件日志位于 Confidential Computing Event Log（CCEL）描述的内存区域，事件摘要按 `MR_new = Hash(MR_old || event_digest)` 扩展到对应 RTMR。日志本身不是可信根；验证方必须回放日志，并把结果与已由 Quote 保护的 RTMR 值比较，才能发现日志是否缺失或被修改。

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

// Protocol Interface Structure
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

### Intel TDX 的寄存器映射

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

下面沿用图 4 的同一条启动路径，按事件发生顺序展开：TDVF 先处理来自 VMM 的输入，UEFI 再度量启动策略和 OS loader，GRUB 最后把链延伸到 kernel 与 initrd。编号 0–18 对应图中的事件节点。

### 度量 TD HOB 与 Configuration FV

[4.2 TD Hand-Off Block（HOB）](https://cdrdv2.intel.com/v1/dl/getContent/733585)和 [3.2 Configuration Firmware Volume（CFV）](https://cdrdv2.intel.com/v1/dl/getContent/733585)由 Host VMM 提供，TD guest 不能默认信任。TDVF 先检查其数据结构，再度量并扩展到 RTMR；度量只能让后续篡改或差异可见，不会把不可信输入变成可信输入。与此同时，TDVF 创建两个 `EFI_CC_EVENT_HOB`，分别携带 TD HOB list 和 CFV 的摘要；DXE 阶段再据此创建 `EFI_CC_EVENT`。

Configuration Firmware Volume 包含所有预置数据，该区域只读。一种可能用途是存放 UEFI Secure Boot 变量内容，例如 PK、KEK、db、dbx。

TD HOB list 用于从 VMM 向 TDVF 传递信息。TD HOB 必须包含 PHIT HOB 和 Resource Descriptor HOB，其他 HOB 可选。

- TD HOB 的第一个 HOB 必须是 PHIT HOB。EfiMemoryTop、EfiMemoryBottom、EfiFreeMemoryTop、EfiFreeMemoryBottom 都应为 0。
- TD HOB 必须至少包含一个 Resource Description HOB，用以声明物理内存资源。

**_0._** `SecTdxHelperLib` 是 TdxHelperLib 的 SEC 实例。它在 SEC 阶段为 TDX 提供以下功能：

- `TdxHelperMeasureTdHob` 度量/扩展 TdHob，并把度量值存入 workarea。
- `TdxHelperMeasureCfvImage` 度量/扩展 Configuration FV 镜像，并把度量值存入 workarea。
- `TdxHelperBuildGuidHobForTdxMeasurement` 为 TDX 度量构建 GuidHob。

### 度量 Secure Boot Policy 到 PCR[7]/RTMR[0]

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

### 度量 Boot Variable 到 PCR[1]/RTMR[0]

_9._ UEFI BootOrder 变量和 Boot#### 变量（仅 device path）。

UEFI 启动相关变量（例如 "BootOrder" 和 "Boot####"）由 TdTcg2Dxe.c 的 `ReadAndMeasureBootVariable()` 度量，事件类型为 [`EV_EFI_VARIABLE_BOOT`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h)。`MeasureAllBootVariables()` 在变量存在时将其度量。

### 选择启动设备之后

_10._ 启动动作 "Calling EFI Application from Boot Option"，表示 Boot Manager 正在尝试从某个启动选项执行代码（**PCR[4]/RTMR[1]**）。

该启动动作由 TdTcg2Dxe.c 的 `OnReadyToBoot()` 度量。调用启动选项之前，度量 "Calling EFI Application from Boot Option"；启动选项返回之后，度量 "Returning from EFI Application from Boot Option"。

_11._ Separator，用以标记离开 pre-boot 环境、进入 post-boot 环境。TCG 模型会在 PCR[0~6] 中写入分隔事件；在这里，RTMR[0] 的分隔事件已于 Secure Boot Policy 度量后写入，因此 `OnReadyToBoot()` 再把分隔事件扩展到 **RTMR[1]**。

_12._ **[可选] 若启用 UEFI Secure Boot**，度量 `EFI_IMAGE_SECURITY_DATABASE` 中用于校验 UEFI 镜像的那一条条目（**PCR[7]/RTMR[0]**）。

启用 UEFI Secure Boot 时，[`DxeImageVerificationLib`](https://github.com/tianocore/edk2/tree/master/SecurityPkg/Library/DxeImageVerificationLib) 依据镜像签名数据库中 [`EFI_SIGNATURE_LIST`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Guid/ImageAuthentication.h) 里的 [`EFI_SIGNATURE_DATA`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Guid/ImageAuthentication.h) 来校验 PE 镜像签名。若用某个 `EFI_SIGNATURE_DATA` 校验了镜像，则 [Measurement.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/Measurement.c) 的 `MeasureVariable()` 会以 [`EV_EFI_VARIABLE_AUTHORITY`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h) 度量该 `EFI_SIGNATURE_DATA`。

_13._ GUID Partition Table（GPT）磁盘布局（**PCR[5]/RTMR[1]**）。

当系统从磁盘中某个 GUID 命名分区启动时，需要度量 GPT 磁盘布局。由 [DxeTpm2MeasureBootLib.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeTpm2MeasureBootLib/DxeTpm2MeasureBootLib.c) 的 `Tcg2MeasureGptTable()` 在 `DxeTpm2MeasureBootHandler()` 中完成。

_14._ 选中的 UEFI 应用代码 PE/COFF 镜像，即 OS loader（**PCR[4]/RTMR[1]**）。

第三方 UEFI 应用（例如 UEFI shell 工具、标准 OS loader 或 OEM 启动选项）由 DxeTpm2MeasureBootLib.c 的 `Tcg2MeasurePeImage()` 在 `DxeTpm2MeasureBootHandler()` 中度量，事件类型为 [`EV_EFI_BOOT_SERVICES_APPLICATION`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h)。若某 UEFI 应用是 DXE 阶段分派的 FV，则无论该 FV 本身是否被度量，都会度量到 PCR4。

### GRUB 把度量链延伸到 OS

[GRUB 2](https://git.savannah.gnu.org/cgit/grub.git/) 接手后，把可信启动链从虚拟固件延伸到 OS。

_注：[shim](https://github.com/rhboot/shim) 的主要职责是把 UEFI Secure Boot 的信任链延伸到 Linux 生态。本文不展开其验证机制，但启用 Secure Boot 时，shim 本身及相关授权事件仍属于启动链的一部分。_

_15._ GRUB 2 度量配置文件（例如 `grub.cfg`）、GRUB 命令、内核二进制、内核命令行和 initrd 二进制（**PCR[8, 9]/RTMR[2]**）。PCR[8] 对应命令行字符串，PCR[9] 对应文件二进制，如下表所示。

为支持机密计算平台上的度量，两个补丁已合入上游：

- [efi/tpm: Add EFI_CC_MEASUREMENT_PROTOCOL support](https://github.com/rhboot/grub2/commit/4c76565b6cb885b7e144dc27f3612066844e2d19)
- [commands/efi/tpm: Re-enable measurements on confidential computing platforms](https://github.com/rhboot/grub2/commit/86df79275d065d87f4de5c97e456973e8b4a649c)

| PCR Index | PCR Usage                                                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 8         | Grub command line: All executed commands (including those from configuration files) will be logged and measured as entered with a prefix of "grub cmd:" |
|           | Kernel command line: Any command line passed to a kernel will be logged and measured as entered with a prefix of "kernel cmdline:"                      |
|           | Module command line: Any command line passed to a kernel module will be logged and measured as entered with a prefix of "module cmdline:"               |
| 9         | Files: Any file read by GRUB will be logged and measured with a descriptive text corresponding to the filename.                                         |

[tpm.c](https://github.com/rhboot/grub2/blob/master/grub-core/commands/tpm.c) 将 `grub_tpm_verify_string()` 和 `grub_tpm_verify_write()` 注册为 `grub_file_verifier`。它们由 [verifiers.c](https://github.com/rhboot/grub2/blob/master/grub-core/kern/verifiers.c) 中的 `grub_verify_string()` 和 `grub_verifiers_open()` 调用。

当 GRUB 2 处理 `GRUB_VERIFY_MODULE_CMDLINE`、`GRUB_VERIFY_KERNEL_CMDLINE`、`GRUB_VERIFY_COMMAND` 等命令类型，或执行 [cmdline.c](https://github.com/rhboot/grub2/blob/master/grub-core/lib/cmdline.c) 中的 `grub_create_loader_cmdline()` 时，会调用 `grub_verify_string()`。随后 `grub_tpm_verify_string()` 调用 `grub_tpm_measure()`，再由 `grub_cc_log_event()` 将字符串度量到 **PCR[8]/RTMR[2]**。

`grub_verifiers_open()` 注册为 [`file.h`](https://github.com/rhboot/grub2/blob/master/include/grub/file.h) 中的 `grub_file_filter` 之一。每当 GRUB 调用 [file.c](https://github.com/rhboot/grub2/blob/master/grub-core/kern/file.c) 中的 `grub_file_open()`，都会触发该过滤器。随后 `grub_tpm_verify_write()` 调用 `grub_tpm_measure()`，再由 `grub_cc_log_event()` 将文件二进制度量到 **PCR[9]/RTMR[2]**。

_16._ 启动动作 "Exit Boot Services Invocation"，表示 Boot Manager 已向 UEFI 发出结束 Boot Services 的调用（**PCR[5]/RTMR[1]**）。

_17._ 启动动作 "Exit Boot Services Returned with Success"，表示 UEFI 成功退出 Boot Services，pre-OS 环境已终止（**PCR[5]/RTMR[1]**）。

ExitBootServices 动作由 TdTcg2Dxe.c 度量。成功时调用 `OnExitBootServices()`，失败时调用 `OnExitBootServicesFailed()`。

### Secure Boot Policy 在启动途中变更

_18._ 平台必须重启，或把这些变量重新度量到 **PCR[7]/RTMR[0]**。另外，对任何 Secure Boot 变量的常规更新都应在 PCR[7] 的首次度量之前，或 `ExitBootServices()` 调用完成之后进行。

UEFI Secure Boot 变量的更新在 Variable [`RuntimeDxe`](https://github.com/tianocore/edk2/tree/master/MdeModulePkg/Universal/Variable/RuntimeDxe) 中度量。如果上述任一 Secure Boot 相关变量被更新，[Measurement.c](https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Universal/Variable/RuntimeDxe/Measurement.c) 的 `MeasureVariable()` 会以 `EV_EFI_VARIABLE_DRIVER_CONFIG` 度量新的数据。

## 远程证明

从验证方视角看，远程证明不是“Quote 签名有效”这一项检查，而是一条三阶段流水线：先确认 Quote 的来源与完整性，再用其中的 MRTD/RTMR 校验事件日志，最后把已验证的事件解释为固件、Secure Boot、GRUB 和内核状态，并与策略及参考值比较。本文不展开 Quote 的生成和传输协议。

### Quote 校验

TDX Quote 携带 TDREPORT 的内容以及支持验证的签名和认证材料。验证方需要检查签名、证书链、吊销状态、TCB 状态和防重放所需的新鲜性（freshness），并确认报告中的平台与 TD 属性满足策略；仅仅成功解析 Quote 并不代表工作负载可信。证明密钥及其认证材料在 IETF RATS 架构中属于背书材料（endorsements）的一部分。具体格式和校验流程见 [Intel SGX Data Center Attestation Primitives（DCAP）](https://github.com/intel/SGXDataCenterAttestationPrimitives)。

### 事件日志回放

事件日志回放是把原始事件日志反序列化后，利用其中的事件重新计算所有度量寄存器的值。每个事件包含一个摘要和度量寄存器索引。验证方创建一组模拟度量寄存器，对每个事件将其摘要扩展到对应的模拟寄存器中。最后将模拟寄存器的值与第一步中 quote 上报的真实度量寄存器值比对。

Canonical 的 TDX 测试仓库提供了 Python 工具 [tdx-tools](https://github.com/canonical/tdx/tree/main/tests/lib/tdx-tools/src/tdxtools)，可在 TD guest 中比较事件日志回放值和 TDREPORT 中的 RTMR。其 `VerifyActor` 展示了三步流程：

- 从 CCEL ACPI 表获取 TD event log。
- 通过 Linux attestation driver 获取 TDREPORT。
- 对比 TDREPORT 中的 RTMR 值与事件日志回放得到的 RTMR 值。

两者完全一致，只能证明事件日志与 TDREPORT 中的 RTMR 自洽；它不证明这些事件代表的组件符合验证方策略。后一个判断属于事件解析和参考值比较。下面摘出当前实现的核心路径：

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
            RTMR(bytearray.fromhex(td_report['td_info']['rtmr_0'])))

        self._verify_single_rtmr(
            1,
            td_event_log_actor.get_rtmr_by_index(1),
            RTMR(bytearray.fromhex(td_report['td_info']['rtmr_1'])))

        self._verify_single_rtmr(
            2,
            td_event_log_actor.get_rtmr_by_index(2),
            RTMR(bytearray.fromhex(td_report['td_info']['rtmr_2'])))

        self._verify_single_rtmr(
            3,
            td_event_log_actor.get_rtmr_by_index(3),
            RTMR(bytearray.fromhex(td_report['td_info']['rtmr_3'])))
```

### 事件解析

事件解析是从事件中提取信息的过程。验证方据此对照评估策略（appraisal policy）、背书材料和参考值（reference values）做出判断。

以 [go-eventlog](https://github.com/google/go-eventlog/blob/main/extract/extract.go) 为例，它可以解析、回放并从多种度量启动日志中提取信息，包括固件、Secure Boot 配置、GRUB 和 Linux kernel 状态。该库明确不负责 Quote 校验，也不提供参考度量值；集成方必须先验证 Quote，再把可信的寄存器值交给日志回放与解析逻辑。下面的 `FirmwareLogState` 代码展示了这类状态提取过程；具体 API 仍可能随该预发布库演进。

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
	sbState, err := SecureBootState(events, registerCfg, opts)
	if err != nil {
		joined = errors.Join(joined, err)
	}
	efiState, err := EfiState(hash, events, registerCfg, opts)

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
		LogType:     registerCfg.LogType,
	}, joined
}
```

### 参考度量值

日志自洽之后，验证方仍需要回答“这些组件是不是预期版本”。这需要参考度量值或更高层的 Reference Integrity Manifest（RIM）。难点在于，最终寄存器值不仅取决于文件内容，还取决于加载后的表示、动态平台数据和事件顺序；不能把几个文件各自做一次 SHA-384 就当作完整参考值。

Canonical 的 [#263](https://github.com/canonical/tdx/issues/263) 记录了离线计算 MRTD/RTMR 时遇到的这些问题。该 issue 于 2026 年 7 月随原 Tech Preview 归档而关闭，并未给出适用于任意 GRUB 启动配置的通用算法。下面是一种作者设想，而不是现有工具已经保证的能力：受 cc-api 仓库中 [cvm-image-rewriter](https://github.com/cc-api/cvm-image-rewriter) 插件架构启发，可以在镜像构建阶段增加 `reference-measurement-calculator`，提取 GRUB、kernel、initrd 和配置，并生成与该构建产物绑定的参考材料。

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

该插件至少需要记录组件摘要、事件类型、目标寄存器和扩展顺序；若实际度量对象是加载后的内存表示，还必须复现对应加载逻辑。它生成的结果应视为特定构建和启动配置的参考材料，而不是天然适用于所有平台的 RTMR 终值。

固件侧可以依据 TDVF Binary Layout 解析镜像并定位 BFV、CFV，具体实现可参考 Contrast 的 [tdx-measure](https://github.com/edgelesssys/contrast/blob/main/tools/tdx-measure/tdvf/tdvf.go)。它展示了如何针对一组明确输入预计算部分度量，但其假设必须与实际 TDVF、QEMU 参数及启动路径一致。

该工具还提供针对给定 firmware、kernel、initrd 和命令行的 RTMR 预计算入口（见 `newRtMrCmd()`）。在输入和事件顺序完全固定时，验证方可以直接比较预计算值与报告值；一旦存在动态 ACPI 数据、启动菜单选择或额外事件，这种比较就容易失效。事件日志因此仍是解释“哪个对象以什么顺序影响了寄存器”的关键证据。

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

- 将*基础* TDVF 特性合入现有的 `OvmfPkgX64.dsc`。（与既有 SEV 对齐）
- 威胁模型：VMM 不排除在 TCB 之外。（不改变现状）
- `OvmfPkgX64.dsc` 包含 SEV/TDX/normal OVMF 的基本启动能力，最终二进制可在 SEV/TDX/normal OVMF 上运行。
- 不修改 OvmfPkgX64 现有镜像布局。
- 无需移除现有特性。
- TD 与 Non-TD 下均不跳过 PEI 阶段。
- 支持基于 RTMR 的度量。
- 度量来自 Host VMM 的外部输入，例如 TdHob、CFV。
- 度量其他外部输入，例如 FW_CFG 数据、os loader、initrd 等。

**构建 TDVF（Config-A）：**

```shell
cd /path/to/edk2
. ./edksetup.sh
make -C BaseTools
build -p OvmfPkg/OvmfPkgX64.dsc -a X64 -t GCC \
  -D CC_MEASUREMENT_ENABLE=TRUE -b RELEASE
```

**Config-B**：

- 在 TDX 专用目录中新增独立的 `IntelTdxX64.dsc`，提供*完整*特性的 TDVF。（与既有 SEV 对齐）
- 威胁模型：VMM 排除在 TCB 之外。（需要必要改动以防御来自 VMM 的攻击）
- `IntelTdxX64.dsc` 包含 TDX/normal OVMF 的基本启动能力，最终二进制可在 TDX/normal OVMF 上运行。
- 将来可能与 AmdSev.dsc 合并，但目前不合并；具体时间未知。只有 Intel 与 AMD 双方都认为各自方案足够成熟时，才会在社区中协调合并。
- 必须新增必要的安全特性作为强制要求，例如基于 RTMR 的可信启动支持。
- 必须度量来自 Host VMM 的外部输入，例如 TdHob、CFV。
- 必须度量其他外部输入，例如 FW_CFG 数据、os loader、initrd 等。
- 跳过 PEI，并拆分 DXE FV，使 TD guest 只加载需要的驱动，以缩小攻击面。

**构建 TDVF（Config-B）：**

```shell
cd /path/to/edk2
. ./edksetup.sh
make -C BaseTools
build -p OvmfPkg/IntelTdx/IntelTdxX64.dsc -a X64 -t GCC -b RELEASE
```

以上配置说明和命令依据 EDK2 的 [TDVF README](https://github.com/tianocore/edk2/blob/master/OvmfPkg/IntelTdx/README.md)；构建前应以目标 EDK2 版本中的同一文件为准。

<span id="jumpB"></span>

### UEFI Boot Phase

UEFI 有六个主要启动阶段，每一阶段都是平台初始化过程中的关键环节。这些阶段合称 Platform Initialization（PI）。

_1._ Security（SEC）

PI 的入口阶段，建立最初的执行环境和临时内存，并把平台启动信息交给 PEI。SEC 可以承载平台信任根，但这不等于 UEFI Secure Boot；后者主要在后续镜像加载时执行签名策略。

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
