---
layout: post
title: "virtCCA: Measured Boot and Attestation Demystified"
date: 2026-01-21
description: An exploration of the measurement and attestation framework within Huawei virtCCA.
tags: security
giscus_comments: true
# related_posts: true
toc: true
---

💭 A quick note: virtCCA's measured-boot and remote-attestation features landed in stages as the product was commercialized, scattering the code across components and fragmenting the documentation. This article consolidates the framework at the architectural level, so readers don't have to dig through scattered older sources.

## Introduction

Huawei virtCCA is a confidential-computing solution built on the TrustZone and Secure EL2 features of the Kunpeng processor. It brings the Arm CCA architectural concepts to existing hardware, giving applications and data a hardware-rooted, verifiable Trusted Execution Environment (TEE) that isolates them from unauthorized access or tampering at runtime.

virtCCA bridges toward native CCA hardware while migrating business applications without modification. Until CCA hardware is widespread, it gives enterprises a practical option with high performance, strong security, and forward compatibility. See [virtCCA: Virtualized Arm Confidential Compute Architecture with TrustZone](https://arxiv.org/abs/2306.11011) and the
[Confidential Computing TEE Suite Technical White Paper](https://www.hikunpeng.com/document/detail/zh/kunpengcctrustzone/tee/cVMcont/kunpengtee_19_0003.html) for details.

As Figure 1 shows, hardware isolation draws the trust boundary, and Measured Boot and Attestation make the state inside it cryptographically verifiable:

- Measured Boot: following "measure before execute", virtCCA hashes the TCB state during platform and CVM (Confidential VM) boot, then stores the results in tamper-resistant registers.

- Attestation: a hardware-protected Attestation Key signs the measurements to produce an Attestation Report, proving the CVM and its platform are genuine and untampered.

Together they build a chain of trust from the hardware Root of Trust (RoT) up to the CVM. A Relying Party can then verify remotely that a workload runs inside a CVM on a genuine virtCCA platform and meets the expected security baseline.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/measurement-and-attestation.png" class="img-fluid rounded z-depth-0 mx-auto d-block" width="80%" zoomable=true %}

<div class="caption">
    Figure 1. Measured Boot and Attestation
</div>

💡 TCB (Trusted Computing Base): the set of hardware, firmware, and software components that must be trusted to keep the system secure. It includes hardware (CPU, RoT), firmware (TF-A, TMM), and critical Guest OS code (UEFI, the GRUB bootloader, the OS kernel). If any TCB component is exploited or modified, the system's confidentiality and integrity no longer hold.

## Measured Boot

The virtCCA architecture consists of the CVM and the underlying virtCCA platform (hardware plus firmware) that hosts it — mirroring the Realm / CCA Platform split in the Arm CCA specification (see [Arm CCA Security Model 1.0](https://developer.arm.com/documentation/DEN0096/latest)). Given this separation, end-to-end Measured Boot comes in two interlocking phases — Platform Measurement and CVM Measurement — shown in Figure 2. Only when the Platform Measurement verifies does the CVM Measurement it carries mean anything.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/platfom-cvm-measurement.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. Platform Measurement and CVM Measurement
</div>

### Platform Measurement

Platform Measurement follows a chained trust-transfer model. During hardware initialization BSBC acts as the Core Root of Trust for Measurement (CRTM); the HSM running on InSE acts as the Root of Trust for Storage (RTS) and the Root of Trust for Reporting (RTR), storing results in InSE's protected internal SRAM (Figure 2). By boot order, virtCCA follows the Arm CCA firmware-boot specification (Trusted Subsystems → Application PE → Monitor → RMM). In practice, the functional-component boundaries differ from Arm CCA because of constraints in the existing Kunpeng hardware, as the table below shows.

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
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Secure boot code embedded in the on-chip ROM</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>ESBC</strong> (External Secure Boot Code)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Secure boot code stored in off-chip non-volatile memory</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>InSE</strong> (Integrated Secure Element)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">An on-chip secure subsystem with physical-security protection</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>HSM</strong> (Hardware Security Module)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">High-security firmware running on InSE; provides key management, secure storage, and Platform-token generation</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>IMU</strong> (Intelligence Management Unit)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Intelligent management unit responsible for hardware management, monitoring, and secure boot</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>IPU</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; color: #888;"><em>TBA</em></td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>ATF</strong> (Arm Trusted Firmware)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Low-level secure firmware running at the highest exception level (EL3)</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>TMM</strong> (TrustZone Management Monitor)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">A lightweight secure-monitoring component responsible for the CVM lifecycle, memory isolation, and interface dispatch</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>UEFI</strong> (Unified Extensible Firmware Interface)</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Platform boot firmware that initializes hardware, performs security checks, and boots the system</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;"><strong>IMF AP</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; color: #888;"><em>TBA</em></td>
      </tr>
    </tbody>
  </table>
</div>

💡 RoTs are components that must be trusted because their misbehavior cannot be detected: (1) the Root of Trust for Measurement (RTM) measures software and forwards the results to the RTS; (2) the Root of Trust for Storage is access-controlled, tamper-resistant storage; (3) the Root of Trust for Reporting reports the RTS's contents — measurement values — as an attestation report; (4) the CRTM is the first code executed after reset and anchors the chain of trust.

### CVM Measurement

virtCCA supports two CVM boot methods (Figure 3): Direct Kernel Boot and Virtual Firmware Boot.

- Direct Kernel Boot: the Hypervisor (QEMU/KVM) loads the OS kernel and the initial RAM filesystem (initramfs) directly, bypassing the virtual firmware and bootloader.
- Virtual Firmware Boot: virtual firmware (UEFI) initializes hardware and loads a bootloader such as GRUB, which then loads the kernel and initramfs. We call this UEFI Boot in the rest of the article.

Direct Kernel Boot starts fast with a small resource footprint, a good fit for lightweight container environments (e.g., Kata Container). UEFI Boot works with existing OS images, handles complex boot configurations such as multi-kernel and multi-boot management, and preserves the cloud-management experience of a regular VM — a good fit for general-purpose VMs (e.g., IaaS cloud instances).

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/direct-kernel-and-firmware-boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 3. Direct Kernel Boot and Firmware Boot
</div>

Both paths follow the same security primitive: Host-provided assets are untrusted. Any input from the Hypervisor (code, data, configuration) must be measured by the TMM — or by an RTM it delegates to — before it is used. The measurements are then extended into specific TMM-protected registers. Per the [Realm Management Monitor specification](https://developer.arm.com/documentation/den0137/latest) (A7.1 Realm measurements), these are the RIM (Realm Initial Measurement) and the REM (Realm Event Measurement). The TMM computes the RIM directly, capturing the initial register and memory state; subsequent RTMs compute the REM — for example, UEFI measuring the GRUB image it loads.

#### Direct Kernel Boot

When a CVM boots via Direct Kernel Boot, virtCCA's measurement logic follows the Arm CCA RMM specification. Since virtCCA's TMM is closed-source, this section uses RMM terminology. The table below summarizes the three phases; for details, see B4.3.9.4 RMI_REALM_CREATE initialization of RIM, B4.3.12.4 RMI_REC_CREATE extension of RIM, and B4.3.1.4 RMI_DATA_CREATE extension of RIM in the [Realm Management Monitor specification](https://developer.arm.com/documentation/den0137/latest).

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

For UEFI-firmware-booted CVMs, the Arm CCA measurement specification did not yet exist at the time, so virtCCA developed its own solution ahead of the standard. Unlike the fixed flow of Direct Kernel Boot, the UEFI boot chain uses diverse boot media and dynamic configuration, making its path highly non-deterministic. Drawing on the trusted-boot mechanisms of TCG and TDX, virtCCA built a deterministic, measurable, and auditable boot pipeline.

🍵 For technical details on Intel TDX's Trusted Boot, see this companion article: [Intel TDX: Measured Boot and Attestation in Grub Boot](https://mahaocheng.me/blog/2025/tdx-measure-boot/).

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/uefi-measured-boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 4. Measurement in UEFI Boot
</div>

As Figure 4 shows, although the UEFI boot combinations are many, the factors that affect the system's security state are relatively fixed. virtCCA measures every security-sensitive element — boot variables, image files, and boot order — uniformly, and builds a reproducible, verifiable Event Log that covers any UEFI boot combination.

Concretely, in the pre-kernel-load environment (spanning the UEFI and bootloader stages), all critical state is measured through EFI_CC_MEASUREMENT_PROTOCOL and extended into REM. The protocol also records each measurement into the CCEL (Confidential Computing Event Log) within the ACPI tables. A CVM can then verify REM by replaying the log, auditing the entire chain of trust end-to-end.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/measured-boot-components-edk2.png" class="img-fluid rounded z-depth-0 mx-auto d-block" width="80%" zoomable=true %}

<div class="caption">
    Figure 5. Measured Boot Components in EDK2
</div>

Figure 5 shows virtCCA's measurement components in UEFI firmware. EFI_CC_MEASUREMENT_PROTOCOL wraps the low-level measurement logic for confidential-computing environments. Per the [UEFI Specification 2.10](https://uefi.org/specs/UEFI/2.10/38_Confidential_Computing.html), it provides hash computation and event-log APIs, and defines the semantic mapping from traditional TPM PCRs to virtCCA REMs (table below). The CcaTcg2Dxe.c driver handles measurement during the DXE phase; the DxeTpm2MeasureBootLib library handles integrity measurement of PE images and the GPT partition table. Source: [src-openeuler/edk2](https://atomgit.com/src-openeuler/edk2/blob/openEuler-24.03-LTS-SP2/edk2.spec), patches 97–106 ("Support measurement when UEFI Boot CVM"). For the flow, see the Measured Boot Flow section of [Intel TDX: Measured Boot and Attestation in Grub Boot](https://mahaocheng.me/blog/2025/tdx-measure-boot/).

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

Under the Delegated Attestation model, several parties share the work of producing the attestation report. As Figure 2 shows, the virtCCA Token wraps two Sub-Tokens: the CVM Token and the Platform Token. The HSM in InSE generates the Platform Token; the TMM caches it in secure memory for performance. The TMM then generates the CVM Token and combines both into the virtCCA Token.

### Authenticity

Each generating entity signs its Sub-Token with a dedicated key, making the chain of trust traceable. The keys derive in a chain: the entity that produced the predecessor Sub-Token derives the next key. To block Sub-Token substitution attacks, Sub-Tokens are cryptographically bound through a "hash anchoring" mechanism — the predecessor Sub-Token's Challenge embeds the hash of the successor Sub-Token's public key.

In virtCCA's key hierarchy: CPAK signs the Platform Token, derived by the HSM from a root key and platform parameters; its public-key certificate is issued by the Huawei Root CA and Sub CA, giving a hardware-rooted identity proof. RAK signs the CVM Token, derived during TMM initialization by the HSM from the root key, the virtCCA platform measurement, and other parameters.

🍵 Since the virtCCA TMM is a closed-source component, the details of how it interacts with the HSM to obtain the RAK and the Platform Token can be found in the [Delegated Attestation Service Integration Guide](https://trustedfirmware-m.readthedocs.io/projects/tf-m-extras/en/latest/partitions/delegated_attestation/delegated_attest_integration_guide.html).

### Attestation Flow

The [virtCCA SDK](https://gitcode.com/openeuler/virtCCA_sdk) is a user-space toolkit for enabling and integrating remote attestation (Figure 6). Its companion [Attestation Demo](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/samples) shows the end-to-end flow: the Server runs inside the CVM and exposes the device certificate (CPAK) and the attestation report (virtCCA Token) through the TSI interface. The Client acts as a Local Verifier, parsing the certificate and report, validating the certificate chain, and evaluating the security policy.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/virtcca-sdk-components.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 6. Component Details in virtCCA SDK
</div>

🍵 For other features — for example Full Disk Encryption, see [attestation/full-disk-encryption](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/full-disk-encryption); for virtCCA's adaptation of [RATS-TLS](https://github.com/inclavare-containers/rats-tls), see [attestation/rats-tls](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/rats-tls).

The diagram below shows the Attestation Demo flow. For real-world deployment and usage, see the feature guide [Enabling Remote Attestation](https://www.hikunpeng.com/document/detail/zh/kunpengcctrustzone/tee/cVMcont/kunpengtee_16_0036.html).

<div id="mermaid-target"></div>

<script src="https://cdn.jsdelivr.net/npm/mermaid@11.12.2/dist/mermaid.min.js"></script>

<script>
document.addEventListener("DOMContentLoaded", function() {
    mermaid.initialize({ startOnLoad: false });

    const graphConfig = `
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
        Note right of C: Input: Reference Values, Endorsements, Appraisal Policy

        C->>C: Verify Token Signature based on Endorsements
        C->>C: Verify Token Claims based on Reference Values

        alt Direct Kernel Boot
            Note right of C: Match RIM against Reference Values
        else UEFI Boot
            Note right of C: Parse & Replay CCEL<br/>Compare Token REMs with Replayed REMs<br/>Extract & Verify Firmware States
        end
        
        C->>C: Apply Appraisal Policy for Final Attestation Result
        C->>S: Attestation Result (Standardized Claims regarding Trustworthiness)
    end
    `;

    const element = document.getElementById('mermaid-target');
    mermaid.render('graph-id-1', graphConfig).then(({ svg }) => {
        element.innerHTML = svg;
    }).catch(err => {
        console.error("Mermaid rendering failed:", err);
        element.innerHTML = "<p style='color:red;'>Mermaid diagram failed to render — please check the console.</p>";
    });
});
</script>

## Summary

Measured boot and remote attestation make for a great "visible trust" story, and everyone loves telling it. For engineers, though, the work really does span the stack — grinding from chip, firmware, and OS up through applications and infrastructure, with one eye on the standards and ecosystem the whole time. After all that, it can be hard to point at exactly where the value lands, and honestly the tech itself isn't all that flashy, 🥲😅.

Anyway, virtCCA has done some of this work, with plenty still to improve. Keep at it, CCA folks.
