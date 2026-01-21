---
layout: post
title: "virtCCA: Measured Boot and Attestation Demystified"
date: 2026-01-21
description: An exploration of the measurement and attestation framework within Huawei virtCCA.
tags: security
categories: virtcca
# giscus_comments: true
# related_posts: true
toc:
  sidebar: left
---

💭 碎碎念：关于 virtCCA 的度量启动和远程证明，其特性随商业化进程分阶段落地，导致相关代码散落在各个组件中，文档也比较碎片化。趁此机会，本文将从架构层面进行统一的梳理与分析，希望能帮大家理清脉络，省去翻阅零散旧资料的周折。

---

## Introduction

Huawei virtCCA 是基于鲲鹏处理器 TrustZone 与 Secure EL2 特性打造的机密计算方案。它在现有硬件上先行实践了 ARM CCA 架构理念，通过构建“基于硬件且可验证”的可行执行环境（TEE），为运行中的应用和数据提供硬件隔离，防止任何未经授权的访问或篡改。

作为向原生 CCA 硬件平滑演进的桥梁，virtCCA 实现了业务应用的零改造迁移。它在 CCA 硬件大规模普及前，为企业提供了兼具高性能、高安全性与前瞻兼容性的实践方案。详见 [virtCCA: Virtualized Arm Confidential Compute Architecture with TrustZone](https://arxiv.org/abs/2306.11011) 与
[机密计算TEE套件技术白皮书](https://www.hikunpeng.com/document/detail/zh/kunpengcctrustzone/tee/cVMcont/kunpengtee_19_0003.html)。

如图 1 所示，硬件隔离机制界定了信任边界，度量启动（Measured Boot）与远程证明（Attestation）通过密码学方式赋予该边界可验证的状态可信：

- 度量启动：遵循“度量先于执行”的原则，在 virtCCA 平台和机密虚拟机（CVM） 的启动过程中，通过哈希度量记录 TCB 的完整性状态，并将度量结果存储到不可篡改寄存器中。

- 远程证明：利用硬件保护的认证密钥（Attestation Key）对度量结果进行数字签名，生成具有证据效力的证明报告（Attestation Report），证明 CVM 及底层平台是真实、安全且未被篡改的。

两者协同配合，构建了从底层硬件信任根（RoT）到上层 CVM 的完整信任链，这使得依赖方（Relying Party）能够远程确信：目标负载运行在真实 virtCCA 平台的 CVM 内，且符合预期安全基准，从而满足机密计算的信任验证需求。

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/measurement-and-attestation.png" class="img-fluid rounded z-depth-0 mx-auto d-block" width="80%" zoomable=true %}

<div class="caption">
    Figure 1. Measured Boot and Attestation
</div>

💡 TCB (Trusted Computing Base) ：为了维持系统安全性而必须被信任的所有硬件、固件及软件组件的集合。这既包含 CPU、RoT 等硬件和 TF-A、TMM 等固件，也涵盖 Guest OS 侧的关键代码，如 UEFI、Bootloader（Grub）以及操作系统内核（Kernel）。如果 TCB 中的任何一个组件出现了漏洞或被恶意篡改，整个系统的安全性（机密性、完整性）都将无法得到保证。

## Measured Boot

virtCCA 架构由 CVM 及其底层支撑环境 virtCCA 平台（硬件与固件集合）共同构成，这在逻辑上映射了 Arm CCA 规范中 Realm 与 CCA Platform 的实体关系（参考 [Arm CCA Security Model 1.0](https://developer.arm.com/documentation/DEN0096/latest) ）。基于这种实体解耦，完整的度量启动由相互衔接的平台度量和 CVM 度量组成，如图 2 所示，也只有当平台度量通过验证，其承载的 CVM 度量才具备实质性的安全意义。

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/platfom-cvm-measurement.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. Platform Measurement and CVM Measurement
</div>

### Platform Measurement

平台度量遵循“链式信任传递”原则，在硬件初始化阶段，BSBC 充当了核心度量信任根 (CRTM) ，运行 HSM 的 InSE 则作为存储信任根（RTS）和报告信任根（RTR），确保度量结果存储于 InSE 内部受保护的 SRAM 中，如图 2 所示。在流程设计上，virtCCA 对标 Arm CCA 的固件引导规范（即 Trusted Subsystems → Application PE → Monitor → RMM）。但在具体的工程实现上，受限于现有鲲鹏硬件的实现架构，相关功能组件的划分与 Arm CCA 存在差异，如下表所示。

<div style="overflow-x: auto;">
  <table style="width: 100%; border-collapse: collapse; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; margin: 24px 0; border: 1px solid #e1e4e8; color: #24292e;">
    <thead>
      <tr style="background-color: #f6f8fa; border-bottom: 2px solid #d1d5da;">
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">Measured Components</th>
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">Descriptions</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>BSBC</strong> (BootROM Secure Boot Code)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">内置于芯片内 ROM 的安全启动代码</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>ESBC</strong> (External Secure Boot Code)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">存储在芯片外非易失性存储的安全启动代码</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>InSE</strong> (Integrated Secure Element)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">芯片内置的安全子系统，具备物理安全防护能力</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>HSM</strong> (Hardware Security Module)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">运行在 InSE 上的高安固件，提供密钥管理、安全存储和 Platform token 生成等功能</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>IMU</strong> (Intelligence Management Unit)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">智能管理单元，用于硬件管理、监控和安全启动</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>IPU</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; color: #888;"><em>TBA</em></td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>ATF</strong> (Arm Trusted Firmware)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">运行在最高特权级 (EL3) 的底层安全固件</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>TMM</strong> (TrustZone Management Monitor)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">轻量安全监控组件，负责 CVM 的生命周期、内存隔离和接口调用</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>UEFI</strong> (Unified Extensible Firmware Interface)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">平台启动固件，负责初始化硬件、执行安全校验并引导系统</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>IMF AP</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; color: #888;"><em>TBA</em></td>
      </tr>
    </tbody>
  </table>
</div>

💡 RoTs 是必须被信任的系统组件，因为无法检测它的非预期行为：（1）度量信任根（RTM）计算软件的度量值，并将度量值发送给 RTS；（2）存储信任根是具备访问控制且防篡改的安全存储；（3）报告信任根上报 RTS 的记录内容，e.g., 度量值，呈现形式是证明报告；（4）CRTM 是系统复位后执行的初始代码，构成信任链的起点。

### CVM Measurement

针对不同的业务场景，virtCCA 为 CVM 提供了两类启动方式：直接内核启动（Direct Kernel Boot）和虚拟固件启动（Virtual Firmware Boot），如图 3 所示。

- 直接内核启动：由 Hypervisor（QEMU/KVM）直接加载操作系统内核（kernel）和初始内存文件系统（initramfs），跳过虚拟固件和启动引导程序。
- 虚拟固件启动：通过虚拟固件（UEFI）初始化硬件，加载启动引导程序（bootloader）如 GRUB，再由 bootloader 加载 kernel 和 initramfs，后文统称为 UEFI 启动。

直接内核启动的优势是启动速度快、资源占用小，适用于轻量化容器环境（如 Kata Container）。UEFI 启动的优势是兼容现有 OS 镜像，支持多内核、多引导管理等复杂启动项，能保持与普通虚拟机一致的云管服务，适用于通用型虚拟机（如 IaaS 云实例）。

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/direct-kernel-and-firmware-boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 3. Direct Kernel Boot and Firmware Boot
</div>

尽管引导路径各异，其底层均遵循统一的安全原语：Host-provided assets are untrusted。任何源自 Hypervisor 侧的输入（包括代码、数据及配置）在被执行或使用前，必须由 TMM 或其委托的 RTM 完成强制性度量。这些度量结果最终被扩展至受 TMM 保护的特定寄存器中。参考 [Realm Management Monitor specification](https://developer.arm.com/documentation/den0137/latest)（A7.1 Realm measurements）的定义，上述寄存器包括 RIM（Realm Initial Measurement）和 REM（Realm Event Measurement），其中 RIM 由 TMM 直接计算，记录初始的寄存器和内存状态，而 REM 则由后续 RTM 计算，例如 UEFI 度量加载的 GRUB。

#### Direct Kernel Boot

采用直接内核启动拉起 CVM 时，virtCCA 的度量逻辑遵循 Arm CCA 的 RMM 规范，由于 virtCCA 的 TMM 是闭源代码，本节将沿用 RMM 语义描述度量流程，下表给出了度量流程的三个阶段，细节可以参考 [Realm Management Monitor specification](https://developer.arm.com/documentation/den0137/latest) 的 B4.3.9.4 RMI_REALM_CREATE initialization of RIM、B4.3.12.4 RMI_REC_CREATE extension of RIM 和 B4.3.1.4 RMI_DATA_CREATE extension of RIM。

<div style="overflow-x: auto;">
  <table style="width: 100%; border-collapse: collapse; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; margin: 24px 0; border: 1px solid #e1e4e8; color: #24292e;">
    <thead>
      <tr style="background-color: #f6f8fa; border-bottom: 2px solid #d1d5da;">
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px; width: 20%;">Phase</th>
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">Description</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; vertical-align: top;"><strong>Creates a Realm</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">
          <div style="margin-bottom: 8px;"><em>CCA Realm <-> virtCCA CVM</em></div>
          <div style="margin-bottom: 8px;"><strong>Command:</strong> <code>RMI_REALM_CREATE</code></div>
          <div style="margin-bottom: 8px;"><strong>Function:</strong> <code>measurement_realm_params_measure()</code></div>
          <strong>Components:</strong>
          <ul style="margin: 4px 0 0 0; padding-left: 18px; line-height: 1.6;">
            <li><strong>flags</strong>: <code>lpa2</code> (LPA2 enabled), <code>sve</code> (SVE enabled), <code>pmu</code> (PMU enabled)</li>
            <li><strong>s2sz</strong>: requested IPA width</li>
            <li><strong>sve_vl</strong>: requested SVE vector length</li>
            <li><strong>num_bps</strong>: number of breakpoints</li>
            <li><strong>num_wps</strong>: number of watchpoints</li>
            <li><strong>pmu_num_ctrs</strong>: requested number of PMU counters</li>
            <li><strong>hash_algo</strong>: algorithm used to measure the initial state of the Realm</li>
          </ul>
        </td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; vertical-align: top;"><strong>Creates a REC</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">
          <div style="margin-bottom: 8px;"><em>REC is a execution context which is associated with Realm vCPU</em></div>
          <div style="margin-bottom: 8px;"><strong>Command:</strong> <code>RMI_REC_CREATE</code></div>
          <div style="margin-bottom: 8px;"><strong>Function:</strong> <code>measurement_rec_params_measure()</code></div>
          <strong>Components:</strong>
          <ul style="margin: 4px 0 0 0; padding-left: 18px; line-height: 1.6;">
            <li><strong>gprs</strong> (general-purpose registers): host provides the initial vaule of registers 0-7, others are zeros</li>
            <li><strong>pc</strong> (program counter): the pc of primary REC is set to <code>LOADER_START_ADDR</code> of Realm, others are zeros</li>
            <li><strong>flags</strong>: <code>runnable</code>, whether REC is eligible for execution, primary REC is set to eligible</li>
          </ul>
        </td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; vertical-align: top;"><strong>Creates a Data Granule</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">
          <div style="margin-bottom: 8px;"><em>Copies contents (e.g., kernel, initramfs) from a Non-secure Granule provided by the caller</em></div>
          <div style="margin-bottom: 8px;"><strong>Command:</strong> <code>RMI_DATA_CREATE</code></div>
          <div style="margin-bottom: 8px;"><strong>Function:</strong> <code>measurement_data_granule_measure()</code></div>
          <strong>Components:</strong>
          <ul style="margin: 4px 0 0 0; padding-left: 18px; line-height: 1.6;">
            <li><strong>ipa</strong>: IPA at which the DATA Granule is mapped in the Realm</li>
            <li><strong>flags</strong>: whether to measure DATA Granule contents</li>
            <li><strong>content</strong>: hash of contents of DATA Granule, or zero if flags indicate DATA Granule contents are unmeasured</li>
          </ul>
        </td>
      </tr>
    </tbody>
  </table>
</div>

#### Virtual Firmware Boot

针对基于 UEFI 固件启动的 CVM 场景，鉴于彼时 Arm CCA 的度量规范仍尚未定义，virtCCA 提前进行了技术预研与方案落地。不同于流程相对固定的直接内核启动，UEFI 启动链条涉及多样的启动介质与动态配置，其路径表现出较强的非确定性。virtCCA 参考了 TCG 与 TDX 的可信启动机制，构建了一套可确定、可度量、可审计的引导链路。

🍵 关于 Intel TDX 的 Trusted Boot 技术细节，可参阅另外一篇文章 [Intel TDX: Measured Boot and Attestation in Grub Boot](https://mahaocheng.me/blog/2025/tdx-measure-boot/)。

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/uefi-measured-boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 4. Measurement in UEFI Boot
</div>

如图 4 所示，尽管 UEFI 的启动组合路径繁杂，但影响系统安全状态的关键因素相对固定。virtCCA 将启动变量、镜像文件以及启动顺序等所有安全敏感状态进行统一度量，通过构建可复现、可验证的事件日志（Event Log），来覆盖任意的 UEFI 启动组合。

具体而言，在内核载入前的预启动环境中（跨越 UEFI 与 bootloader 阶段），所有关键状态均通过 EFI_CC_MEASUREMENT_PROTOCOL 进行度量，并扩展到 REM 中。同时，该协议将每次度量操作记录至 ACPI 表中的 CCEL（Confidential Computing Event Log）。这种设计支持 CVM 通过日志重放机制验证 REM 的真实性，从而确保了整条信任链的端到端可审计。

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/measured-boot-components-edk2.png" class="img-fluid rounded z-depth-0 mx-auto d-block" width="80%" zoomable=true %}

<div class="caption">
    Figure 5. Measured Boot Components in EDK2
</div>

virtCCA 在 UEFI 固件中的度量组件如图 5 所示。其中，EFI_CC_MEASUREMENT_PROTOCOL 封装了机密计算环境下的底层度量逻辑。遵循 [UEFI Specification 2.10](https://uefi.org/specs/UEFI/2.10/38_Confidential_Computing.html) 规范，该实例提供哈希计算和事件日志的能力，并确立了从传统的 TPM PCR 到 virtCCA REM 的语义映射（见下表）。CcaTcg2Dxe.c 驱动负责处理 DXE 阶段的度量事务，DxeTpm2MeasureBootLib 库则负责 PE 镜像及 GPT 分区表的完整性度量，代码可参阅 [src-openeuler/edk2](https://atomgit.com/src-openeuler/edk2/blob/openEuler-24.03-LTS-SP2/edk2.spec) 的 patch97-patch106（Support measurement when UEFI Boot CVM），流程可参考 [Intel TDX: Measured Boot and Attestation in Grub Boot](https://mahaocheng.me/blog/2025/tdx-measure-boot/) 的 Measured Boot Flow 章节。

<div style="overflow-x: auto;">
  <table style="width: 100%; border-collapse: collapse; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; margin: 24px 0; border: 1px solid #e1e4e8; color: #24292e;">
    <thead>
      <tr style="background-color: #f6f8fa; border-bottom: 2px solid #d1d5da;">
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">TPM PCR</th>
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">TDX</th>
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">virtCCA/CCA</th>
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">Typical Usage of Measurement Registers</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">0</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">MRTD</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RIM</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Virtual firmware code (match PCR[0])</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">1, 7</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RTMR[0]</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">REM0</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Firmware configuration (match PCR[1,7])</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">2~6</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RTMR[1]</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">REM1</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Firmware loaded component, such as OS loader (match PCR[4,5])</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">8~15</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RTMR[2]</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">REM2</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">OS component, such as OS kernel, initrd, and application (match PCR[8~15]), the usage is OS dependent</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">16-...</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RTMR[3]</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">REM3</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Reserved for special usage only</td>
      </tr>
    </tbody>
  </table>
</div>

## Attestation

在 Delegated Attestation 模型下，证明报告的生成职责由多个参与方协同承担。如图 2 所示，virtCCA Token 封装了两个 Sub-Token（CVM Token 与 Platform Token）。其中，Platform Token 由 InSE 的 HSM 生成，出于性能考虑会缓存到 TMM 的安全内存；TMM 则在生成 CVM Token 基础上组合两者生成 virtCCA Token。

### Authenticity

为了确保信任链的可追溯性，每个 Sub-Token 均由对应生成实体使用专用密钥进行签名。这些密钥遵循链式派生逻辑：链中的每个密钥都由其前序 Sub-Token 的生成实体派生。为防止 Sub-Token 替换攻击，各个 Sub-Token 通过“哈希锚定”机制实现密码学绑定，在前序 Sub-Token 的 Challenge 中嵌入后序 Sub-Token 的公钥哈希。

具体到 virtCCA 的密钥体系，CPAK 是 Platform Token 的签名密钥，由 HSM 基于根密钥及平台参数派生，其公钥证书由 Huawei Root CA 及 Sub CA 签发背书，提供了硬件级身份证明；RAK 是 CVM Token 的签名密钥，派生于 TMM 初始化阶段，由 HSM 结合根密钥和 virtCCA 平台度量值等参数生成。

🍵 鉴于 virtCCA TMM 属于闭源组件，关于其与 HSM 交互以获取 RAK 和 Platform Token 的流程细节，可参阅 [Delegated Attestation Service Integration Guide](https://trustedfirmware-m.readthedocs.io/projects/tf-m-extras/en/latest/partitions/delegated_attestation/delegated_attest_integration_guide.html)。

### Attestation Flow

virtCCA 提供了用户态工具包 [virtCCA SDK](https://gitcode.com/openeuler/virtCCA_sdk)，用于简化远程证明的使能和集成，如图 6 所示。其配套的 [Attestation Demo](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/samples) 展示了端到端的证明逻辑：Server 端在 CVM 运行时环境里，利用 TSI 接口向上透传设备证书（CPAK 证书）与证明报告（virtCCA Token）。Client 端则充当 Local Verifier 角色，解析收到的证书与报告，进行证书链校验和安全策略评估。

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/virtcca-sdk-components.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 6. Component Details in virtCCA SDK
</div>

🍵 其他特性，例如 Full Disk Encryption 可参阅 [attestation/full-disk-encryption](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/full-disk-encryption)；virtCCA 对 [RATS-TLS](https://github.com/inclavare-containers/rats-tls) 的适配可参阅 [attestation/rats-tls](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/rats-tls)。

如下展示了 Attestation Demo 的具体流程，实际部署与使用请参阅 [使能远程证明](https://www.hikunpeng.com/document/detail/zh/kunpengcctrustzone/tee/cVMcont/kunpengtee_16_0036.html) 的特性指南。

<div class="mermaid">
sequenceDiagram
    autonumber
    
    participant C as Verifier (Client)
    participant S as Attester (Server)
    participant SDK as TSI SDK / Driver

    Note over S, SDK: [ TEE / Confidential VM Boundary ]

    rect rgb(240, 244, 255)
        Note over C, SDK: Phase 1: Evidence & Identity Collection
        C->>S: Request Identity Certificate (CPAK)
        S->>SDK: get_dev_cert()
        SDK-->>S: Return CPAK Certificate
        S-->>C: Deliver CPAK Certificate

        C->>C: Generate Fresh Nonce (Challenge)

        C->>S: Request Attestation Evidence (virtCCA Token)
        S->>SDK: get_attestation_token(nonce)
        SDK-->>S: Return Signed Evidence
        S-->>C: Deliver Signed Evidence

        alt UEFI Boot
        C->>S: Request Event Log (CCEL)
        S-->>C: Send CCEL Data
        end
    end

    rect rgb(255, 251, 235)
        Note over C: Phase 2: Appraisal & Verification
        Note right of C: Input: Reference Values, Endorsements, Verification Policy

        C->>C: Validate Certificate Chain and Signature
        C->>C: Compare Freshness and Hash Binding
        C->>C: Evaluate Platform and CVM Token Details

        alt Direct Kernel Boot
            Note right of C: Match RIM against Reference Values
        else UEFI Boot
            Note right of C: Parse & Replay CCEL<br/>Compare Token REMs with Replayed REMs<br/>Extract & Verify Firmware States
        end

        C->>S: 3. Final Attestation Result (Success/Fail)
    end

</div>

## Summary

度量启动和远程证明这玩意儿，大家一直聊得挺热闹，谁都想讲好“看得见的信任”这个故事。但对技术人来说，确实是全方位的折腾：从最底下的芯片、固件、OS 一路磨到应用和基建，中间还得兼顾标准和生态。等真的忙活完一通，有时也说不清价值到底体现在哪，说到底技术本身也没多炫酷，🥲😅。

总之，virtCCA 目前在这块做了一些工作，还有很多不完善的地方，大家 CCA 加油吧。

<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad:true});</script>
