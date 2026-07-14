---
layout: post
title: "virtCCA: Measured Boot and Attestation Demystified"
date: 2026-01-21
description: How Huawei virtCCA measures its platform and confidential VMs for remote attestation.
tags: security
giscus_comments: true
# related_posts: true
toc: true
---

💭 A quick note: virtCCA's measured-boot and remote-attestation features were added gradually as the product matured. The implementation now spans several components, while the documentation is scattered across sources written at different times. This article brings those pieces together into one architectural view.

## Introduction

Huawei virtCCA is a confidential-computing system built on the TrustZone and Secure EL2 features of Kunpeng processors. It implements the main ideas of Arm CCA on existing hardware: applications run in a hardware-isolated Trusted Execution Environment (TEE) whose state can be verified.

virtCCA also provides a migration path toward native CCA hardware without requiring changes to existing applications. See [virtCCA: Virtualized Arm Confidential Compute Architecture with TrustZone](https://arxiv.org/abs/2306.11011) and the
[Confidential Computing TEE Suite Technical White Paper](https://www.hikunpeng.com/document/detail/zh/kunpengcctrustzone/tee/cVMcont/kunpengtee_19_0003.html) for details.

Hardware isolation draws the trust boundary. Measured Boot and Attestation make the state within that boundary verifiable, as shown in Figure 1:

- Measured Boot: following the rule "measure before execute," virtCCA hashes the state of its Trusted Computing Base (TCB) while the platform and Confidential VM (CVM) boot. It records the resulting measurements in protected registers.

- Attestation: a hardware-protected Attestation Key signs those measurements and places them in an Attestation Report. A verifier uses the report to establish the identity and measured state of the CVM and its platform.

The result is a chain of trust from the hardware Root of Trust (RoT) to the CVM. A Relying Party can check remotely that a workload is running in a CVM on a genuine virtCCA platform and that its measurements match an accepted security baseline.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/measurement-and-attestation.png" class="img-fluid rounded z-depth-0 mx-auto d-block" width="80%" zoomable=true %}

<div class="caption">
    Figure 1. Measured Boot and Attestation
</div>

💡 The Trusted Computing Base (TCB) is the hardware, firmware, and software that the system depends on for security. In virtCCA it includes hardware such as the CPU and RoT, firmware such as TF-A and the TMM, and critical guest code such as UEFI, GRUB, and the OS kernel. A vulnerability or unauthorized change in any of these components can break the system's confidentiality or integrity.

## Measured Boot

The virtCCA architecture separates a CVM from the underlying hardware and firmware that host it. This corresponds to the distinction between a Realm and the CCA Platform in the Arm CCA specification (see [Arm CCA Security Model 1.0](https://developer.arm.com/documentation/DEN0096/latest)). Measured Boot therefore has two linked parts, shown in Figure 2: Platform Measurement and CVM Measurement. A CVM measurement is useful only when the platform reporting it can also be trusted.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/platfom-cvm-measurement.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. Platform Measurement and CVM Measurement
</div>

### Platform Measurement

Platform Measurement passes trust from one boot stage to the next. During hardware initialization, the BSBC serves as the Core Root of Trust for Measurement (CRTM). The HSM running on the InSE provides the Root of Trust for Storage (RTS) and Root of Trust for Reporting (RTR), with measurements held in the InSE's protected internal SRAM (Figure 2). The boot order follows the Arm CCA firmware model: Trusted Subsystems → Application PE → Monitor → RMM. The component boundaries differ from native Arm CCA because virtCCA works within the architecture of existing Kunpeng hardware.

<div style="overflow-x: auto;">
  <table style="width: 100%; border-collapse: collapse; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; margin: 24px 0; border: 1px solid #e1e4e8; color: #24292e;">
    <thead>
      <tr style="background-color: #f6f8fa; border-bottom: 2px solid #d1d5da;">
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">Measured Components</th>
        <th style="padding: 12px 16px; border: 1px solid #e1e4e8; text-align: left; font-size: 14px;">Description</th>
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

💡 Roots of Trust (RoTs) are components whose incorrect behavior cannot be detected by another component in the same trust model. The Root of Trust for Measurement (RTM) measures software and sends the values to the RTS. The Root of Trust for Storage keeps those values in access-controlled, tamper-resistant storage. The Root of Trust for Reporting turns the stored measurements into an attestation report. The CRTM is the first code to run after reset and starts this chain.

### CVM Measurement

virtCCA supports two ways to boot a CVM (Figure 3): Direct Kernel Boot and Virtual Firmware Boot.

- Direct Kernel Boot: the hypervisor (QEMU/KVM) loads the OS kernel and initial RAM filesystem (initramfs) directly, without virtual firmware or a bootloader.
- Virtual Firmware Boot: virtual firmware (UEFI) initializes the virtual hardware and loads a bootloader such as GRUB; the bootloader then loads the kernel and initramfs. The rest of this article calls this path UEFI Boot.

Direct Kernel Boot starts quickly and uses fewer resources, which suits lightweight container environments such as Kata Containers. UEFI Boot works with existing OS images and more complex configurations, including multiple kernels and boot entries. It also behaves more like a conventional VM from the cloud management layer, making it a better fit for general-purpose IaaS instances.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/direct-kernel-and-firmware-boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 3. Direct Kernel Boot and Firmware Boot
</div>

Both paths start from the same security rule: assets supplied by the host are untrusted. Before the CVM uses host-supplied code, data, or configuration, the TMM—or an RTM to which it delegates the work—must measure it. Those measurements are extended into registers protected by the TMM. Section A7.1 of the [Realm Management Monitor specification](https://developer.arm.com/documentation/den0137/latest) defines two kinds of register: the Realm Initial Measurement (RIM) and Realm Event Measurements (REMs). The TMM calculates the RIM from the CVM's initial register and memory state. Later boot stages extend the REMs; for example, UEFI measures the GRUB image before loading it.

#### Direct Kernel Boot

For Direct Kernel Boot, virtCCA follows the measurement model in the Arm CCA RMM specification. The TMM is closed source, so the description below uses the corresponding RMM terminology. The three operations are summarized in the table. Their definitions appear in sections B4.3.9.4 (`RMI_REALM_CREATE` initialization of RIM), B4.3.12.4 (`RMI_REC_CREATE` extension of RIM), and B4.3.1.4 (`RMI_DATA_CREATE` extension of RIM) of the [Realm Management Monitor specification](https://developer.arm.com/documentation/den0137/latest).

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
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; vertical-align: top;"><strong>Create a Realm</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">
          <div style="margin-bottom: 8px;"><em>A CCA Realm corresponds to a virtCCA CVM.</em></div>
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
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; vertical-align: top;"><strong>Create a REC</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">
          <div style="margin-bottom: 8px;"><em>A REC is the execution context associated with a Realm vCPU.</em></div>
          <div style="margin-bottom: 8px;"><strong>Command:</strong> <code>RMI_REC_CREATE</code></div>
          <div style="margin-bottom: 8px;"><strong>Function:</strong> <code>measurement_rec_params_measure()</code></div>
          <strong>Components:</strong>
          <ul style="margin: 4px 0 0 0; padding-left: 18px; line-height: 1.6;">
            <li><strong>gprs</strong> (general-purpose registers): the host supplies the initial values of registers 0–7; the remaining registers are zeroed</li>
            <li><strong>pc</strong> (program counter): the primary REC starts at the Realm's <code>LOADER_START_ADDR</code>; the PCs of other RECs are zeroed</li>
            <li><strong>flags</strong>: includes <code>runnable</code>, which indicates whether a REC may execute; the primary REC is runnable</li>
          </ul>
        </td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px; vertical-align: top;"><strong>Create a Data Granule</strong></td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">
          <div style="margin-bottom: 8px;"><em>Copies content such as the kernel or initramfs from a Non-secure Granule supplied by the caller.</em></div>
          <div style="margin-bottom: 8px;"><strong>Command:</strong> <code>RMI_DATA_CREATE</code></div>
          <div style="margin-bottom: 8px;"><strong>Function:</strong> <code>measurement_data_granule_measure()</code></div>
          <strong>Components:</strong>
          <ul style="margin: 4px 0 0 0; padding-left: 18px; line-height: 1.6;">
            <li><strong>ipa</strong>: IPA at which the DATA Granule is mapped in the Realm</li>
            <li><strong>flags</strong>: indicates whether to measure the DATA Granule's contents</li>
            <li><strong>content</strong>: the hash of the DATA Granule's contents, or zero when the flags mark those contents as unmeasured</li>
          </ul>
        </td>
      </tr>
    </tbody>
  </table>
</div>

#### Virtual Firmware Boot

When virtCCA added support for UEFI Boot, Arm CCA had not yet defined a measurement specification for this path. virtCCA therefore designed its own scheme, drawing on the trusted-boot mechanisms used by TCG and TDX. UEFI Boot is harder to measure than Direct Kernel Boot because its boot media and configuration can vary. The scheme has to preserve evidence of those choices without assuming a single fixed sequence.

🍵 For technical details on Intel TDX's Trusted Boot, see this companion article: [Intel TDX: Measured Boot and Attestation in Grub Boot](https://mahaocheng.me/blog/2025/tdx-measure-boot/).

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/uefi-measured-boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 4. Measurement in UEFI Boot
</div>

The possible UEFI boot paths are numerous, but the inputs that determine their security state fall into a smaller set. virtCCA measures boot variables, image files, boot order, and other security-sensitive inputs, then records the measurements in an Event Log. This gives the verifier a reproducible account of the path that was actually taken rather than requiring every valid path to be enumerated in advance.

Before the kernel is loaded, UEFI and the bootloader measure critical state through `EFI_CC_MEASUREMENT_PROTOCOL` and extend the results into the REMs. The protocol records the same events in the Confidential Computing Event Log (CCEL), exposed through an ACPI table. A verifier can replay that log, reconstruct the REM values, and compare them with the values in the attestation token. Matching values show that the log corresponds to the measured boot sequence; policy evaluation still determines whether that sequence is acceptable.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/measured-boot-components-edk2.png" class="img-fluid rounded z-depth-0 mx-auto d-block" width="80%" zoomable=true %}

<div class="caption">
    Figure 5. Measured Boot Components in EDK2
</div>

Figure 5 shows the components that implement measurement in virtCCA's UEFI firmware. `EFI_CC_MEASUREMENT_PROTOCOL` provides the hashing and event-log interface for a confidential-computing guest. Following the [UEFI Specification 2.10](https://uefi.org/specs/UEFI/2.10/38_Confidential_Computing.html), the implementation maps the conventional uses of TPM PCRs to virtCCA REMs, as summarized below. The `CcaTcg2Dxe.c` driver handles DXE-phase measurements, while `DxeTpm2MeasureBootLib` measures PE images and the GPT partition table. The implementation is carried by patches 97–106 ("Support measurement when UEFI Boot CVM") in [src-openeuler/edk2](https://atomgit.com/src-openeuler/edk2/blob/openEuler-24.03-LTS-SP2/edk2.spec). The Measured Boot Flow section of [Intel TDX: Measured Boot and Attestation in Grub Boot](https://mahaocheng.me/blog/2025/tdx-measure-boot/) describes the analogous flow in more detail.

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
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Virtual firmware code (corresponds to PCR[0])</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">1, 7</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RTMR[0]</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">REM0</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Firmware configuration (corresponds to PCR[1,7])</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">2~6</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RTMR[1]</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">REM1</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Components loaded by firmware, such as the OS loader (corresponds to PCR[4,5])</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">8~15</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RTMR[2]</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">REM2</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">OS components such as the kernel, initrd, and applications (corresponds to PCR[8–15]); exact usage depends on the OS</td>
      </tr>
      <tr>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">16+</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">RTMR[3]</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">REM3</td>
        <td style="padding: 12px 16px; border: 1px solid #e1e4e8; font-size: 14px;">Reserved for special-purpose use</td>
      </tr>
    </tbody>
  </table>
</div>

## Attestation

virtCCA uses delegated attestation: the platform and the CVM each produce part of the evidence instead of relying on one component to report on the entire stack. As Figure 2 shows, a virtCCA Token contains two sub-tokens, the Platform Token and the CVM Token. The HSM in the InSE generates the Platform Token, which the TMM caches in secure memory. The TMM generates the CVM Token and packages the two together.

### Authenticity

Each component signs its own sub-token with a dedicated key. The keys form a derivation chain: the component responsible for the preceding sub-token derives the key used for the next one. The sub-tokens must also be bound to each other; otherwise, an attacker could replace one with evidence from another system. virtCCA prevents this by placing the hash of the successor's public key in the predecessor sub-token's Challenge field.

Two keys matter here. The CCA Platform Attestation Key (CPAK) signs the Platform Token. The HSM derives it from a root key and platform parameters, and Huawei's Root CA and subordinate CA issue the corresponding certificate. The Realm Attestation Key (RAK) signs the CVM Token. During TMM initialization, the HSM derives the RAK from the root key, the virtCCA platform measurement, and other parameters.

🍵 The virtCCA TMM is closed source, so its exact interaction with the HSM is not publicly visible. The [Delegated Attestation Service Integration Guide](https://trustedfirmware-m.readthedocs.io/projects/tf-m-extras/en/latest/partitions/delegated_attestation/delegated_attest_integration_guide.html) describes the corresponding integration pattern for obtaining a RAK and Platform Token.

### Attestation Flow

The [virtCCA SDK](https://gitcode.com/openeuler/virtCCA_sdk) provides the user-space interfaces needed to integrate remote attestation (Figure 6). Its [Attestation Demo](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/samples) shows the complete exchange. A server inside the CVM obtains the device certificate and virtCCA Token through the TSI interface and returns them to a client. The client is the Local Verifier: it parses the evidence, validates the certificate chain and token signatures, compares the measurements with reference values, and applies its appraisal policy.

{% include figure.liquid path="assets/img/2026-01-21-virtcca-measured-boot-attestation/virtcca-sdk-components.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 6. Component Details in virtCCA SDK
</div>

🍵 The SDK repository also contains examples for [Full Disk Encryption](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/full-disk-encryption) and virtCCA's integration with [RATS-TLS](https://gitcode.com/openeuler/virtCCA_sdk/tree/master/attestation/rats-tls).

The sequence below follows the Attestation Demo. For deployment instructions, see [Enabling Remote Attestation](https://www.hikunpeng.com/document/detail/zh/kunpengcctrustzone/tee/cVMcont/kunpengtee_16_0036.html).

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

Measured boot and remote attestation are often presented as a neat story about making trust visible. The engineering is less neat. It stretches from the chip through firmware and the OS to applications and infrastructure, while standards and the surrounding ecosystem continue to evolve. After all that work, the visible result is simply evidence that a verifier can compare with reference values and policy. It may not look exciting, but producing trustworthy evidence across the whole stack takes a great deal of work. 🥲😅

virtCCA has implemented part of that path, though there is still plenty to improve. Keep going, CCA folks.
