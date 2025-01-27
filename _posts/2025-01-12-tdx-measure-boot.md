---
layout: post
title: Measured boot in Intel TDX's grub boot
date: 2025-01-12
description: How to build a trusted chain when launch TD guest using grub boot.
tags: tdx measurement attestation
categories: cc
# giscus_comments: true
# related_posts: true
toc:
  sidebar: left
---

Intel Trust Domain Extension (TDX) allows people to deploy hardware-isolated virtual machines (VMs) called trust domains (TDs). A TD VM is isolated from the virtual machine manager (VMM), hypervisor, and other non-TD software on the host platform. The entire memory contents of a TD is encrypted using a multiple-key encryption method. Intel TDX excludes the host platform from the TD's trusted computing base (TCB). 

Intel TDX workloads are primarily VM images that tenants want to run as TDs in a cloud environment. Special Open Virtual Machine Firmware (OVMF) supplied by the CSP is required to run a VM in a TD, such as TDX Virtual Firmware (TDVF). TDVF is an EDK2 based project to enable UEFI support for TDX based Virtual Machines. It provides the capability to launch a TD. TDVF allows two configurations with different features, please see [Appendix: TDVF Configurations](#jumpA) from [Configurations and Features](https://github.com/tianocore/edk2/blob/master/OvmfPkg/IntelTdx/README.md).

During TD boot, the hash-chained measurement on TCB will be extended to some secure registers (also known as measured boot). The values from several secure registers construct to a report and are finally signed to be a quote by an attestation 
key. Before providing data to the workload, a relying party uses attestation to verify that the software is running within a TD on a genuine Intel TDX system and at the specified security level.

*NOTE: Here TCB refers to all of a system's hardware, firmware, and software components that provide a secure environment, including hardware information such as CPU, SEAM firmware, and guest components such as OVMF, bootloader (shim/grub), and kernel.*

**The above is a brief description of Intel TDX. In the upcoming series of articles, we will focus on TD's measured boot and attestation.**

## TD Boot Types

The following diagram illustrates the TD boot type and boot process.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/td_guest_boot_process.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 1. TD Guest Boot Process
</div>

**Direct boot** is a boot process where the system boots directly into the OS without an intermediate boot loader. It can also be referred to as direct kernel boot with firmware, in which firmware loads kernel and initrd through the FwCfg device provided by Qemu. 

**Grub boot** involves using the Grub bootloader, which provides advanced boot menu options, allowing you to select different operating systems and customize boot configurations. It can also be referred to as firmware-only boot, in which firmware launches a bootloader from a disk image.

The detailed boot flow for different TD boot methods can be found in Figure 2.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/detailed_flow_for_different_td_boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. Detailed Flow for Different TD Boot
</div>

**Today we will be diving into Measured Boot in Grub Boot.** 

In the virtual firmware, i.e., OVMF, the image handler `DxeTpmMeasureBootHandler` will be triggered when loading EFI image via `CoreLoadImageCommon()`. The `DxeTpmMeasureBootHandler` measures the objects like FV, QEMU CFG, VMM Hob, Variable into measurement registers (MRs). Then in boot loader ShimX64.efi, `TpmMeasureVariable()` measures the secure boot’s certificates into MRs. If secure boot is not enabled, ShimX64.efi may be absent from the VM image. Finally, boot loader GrubX64.efi measures kernel binary and cmdline, initrd binary, and grub’s module into MRs. 

Yes, the process here is draws on the TCG trusted boot chain. The difference is that it does not trust the hypervisor and avoids using mutable non-volatile storage (causing MR change). Futhermore, its trust chain can trace back to the TDX enable hardware.

## UEFI Boot Sequence

UEFI allows the extension of platform firmware by loading UEFI driver and UEFI application images. When UEFI drivers and UEFI applications are loaded they have access to all UEFI-defined runtime and boot services. See the Booting Sequence Figure 3 below.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/uefi_booting_sequence.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 3. UEFI Booting Sequence
</div>

UEFI allows the consolidation of boot menus from the OS loader and platform firmware into a single platform firmware menu. These platform firmware menus will allow the selection of any UEFI OS loader from any partition on any boot medium that is supported by UEFI boot services.

The OS loader operates as a UEFI application, utilizing UEFI services via Boot Services (BS) and Runtime Services (RT). It then invokes `ExitBootServices()` to terminate BS and release its resources, with only RT remaining available to the OS.

A detailed introduction to UEFI boot phases refers to [Appendix: UEFI Boot Phase](#jumpB) from [UEFI boot phases](https://secret.club/2020/05/26/introduction-to-uefi-part-1.html#uefi-boot-phases). 

## Measured Boot Component

See Figure 4, the pre-boot environment before the kernel includes the TDVF/OVMF phase and the bootloader phase (shim and grub). The whole boot chain will be measured into Runtime Measurement Register (RTMR) via `EFI_CC_MEASUREMENT_PROTOCOL`.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/td_measurement_process.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 4. TD Measurement Process
</div>

Similar to the TCG event log, `EFI_CC_MEASUREMENT_PROTOCOL` logs the events into confidential computing event log (CCEL) ACPI table and the measurement hash is extended to the corresponding RTMR register. The event logs in CCEL table can be replayed within a TD guest to verify the RTMR value.

The Measured Boot Component in EDK2 is as follows.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/measured_boot_component_in_edk2.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 5. Measured Boot Component in EDK2
</div>

The [`SecTdxHelperLib`](https://github.com/tianocore/edk2/tree/master/OvmfPkg/IntelTdx/TdxHelperLib) library provides measurement functions in SEC phase. 
The [`EFI_CC_MEASUREMENT_PROTOCOL`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Protocol/CcMeasurement.h)  protocol abstracts the confidential computing (CC) measurement operation in UEFI guest environment.
The [TdTcg2Dxe.c](https://github.com/tianocore/edk2/blob/master/OvmfPkg/Tcg/TdTcg2Dxe/TdTcg2Dxe.c) DXE driver handles the DXE phase measurement. 
The [`DxeTpm2MeasureBootLib`](https://github.com/tianocore/edk2/tree/master/SecurityPkg/Library/DxeTpm2MeasureBootLib) library handles the PE image measurements and GPT measurement. All event type definition can be found at [`UefiTcgPlatform.h`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h). 

Here we give a description of `EFI_CC_MEASUREMENT_PROTOCOL` from [Confidential Computing in UEFI Specification](https://uefi.org/specs/UEFI/2.10/38_Confidential_Computing.html#).

If a virtual firmware with CC capability supports measurement, the virtual firmware should produce `EFI_CC_MEASUREMENT_PROTOCOL` with new GUID `EFI_CC_MEASUREMENT_PROTOCOL_GUID` to report event log and provide hash capability. 

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

This protocol defines four parameters in the above interface structure, including: 

* GetCapability provides protocol capability information and state information. 
* GetEventLog allows a caller to retrieve the address of a given event log and its last entry. 
* HashLogExtendEvent provides callers with an opportunity to extend and optionally log events without requiring knowledge of actual CC command. 
* MapPcrToMrIndex provides callers information on TPM PCR to CC MR mapping. 

**Mapping for Intel TDX**

The following table shows the TPM Platform Configuration Register (PCR) index mapping and CC event log MR index interpretation for Intel TDX, where MRTD means Trust Domain Measurement Register and RTMR means Runtime Measurement Register. There certainly are fewer TD MRs than TPM PCRs. They are typically mapped as below:

| TPM PCR Index | Typical Usage of Measurement Registers                | TDX MR Index                   |
| ------------- | ----------------------------------------------------- | ------------------------------ |
| 0             | FirmwareCode (BFV, including init page table)         | MRTD                           |
| 1             | FirmwareData (CFV, TD Hob, ACPI Table, Boot Variable) | RTMR[0]                        |
| 2             | Option ROM code                                       | RTMR[1]                        |
| 3             | Option ROM code                                       | RTMR[1]                        |
| 4             | OS loader code                                        | RTMR[1]                        |
| 5             | GUID partition table (GPT)                            | RTMR[1]                        |
| 6             | N/A                                                   | N/A                            |
| 7             | Secure Boot Configuration                             | RTMR[0]                        |
| 8~15          | TD OS measurement                                     | RTMR[2]                        |

*NOTE: RTMR[3] is reserved for special usage, such as virtual TPM. Users have the flexibility to utilize RTMR[3] if it is not required for these specialized purposes.*

The typical usage of MRTD and RTMR is shown below, more detailes could be found in [8.1 Measurement Register Usage in TD](https://cdrdv2.intel.com/v1/dl/getContent/733585).
- MRTD is for the TDVF code (match PCR[0]).
- RTMR[0] is for the TDVF configuration (match PCR[1,7]). The usage should follow TCG Platform Firmware Profile (PFP) specification.
- RTMR[1] is for the TDVF loaded component, such as OS loader (match PCR[4,5]). The usage should follow TCG Platform Firmware Profile (PFP) specification.
- RTMR[2] is for the OS component, such as OS kernel, initrd, and application (match PCR[8~15]). The usage is OS dependent.
- RTMR[3] is reserved for special usage only.

## Measured Boot Flow

In general, the transitions (dotted line in Booting Sequence figure) where events are measured. Hence a trusted chain, i.e., virtual firmware -> bootloaders -> OS -> applications will be built.

**Measure TdHob and Configuration FV (Cfv)**

 [4.2 TD Hand-Off Block (HOB)](https://cdrdv2.intel.com/v1/dl/getContent/733585) and [3.2 Configuration Firmware Volume (CFV)](https://cdrdv2.intel.com/v1/dl/getContent/733585) are external data provided by Host VMM. These are not trusted in TD guest. So they should be validated, measured and extended to TD RTMR registers. In the meantime 2 `EFI_CC_EVENT_HOB` are created. These 2 GUIDed HOBs carry the hash value of TdHobList and Configuration FV. In DXE phase `EFI_CC_EVENT` can be created based on these 2 GUIDed HOBs.

Configuration Firmware Volume includes all the provisioned data. This region is read only. One possible usage is to provide UEFI Secure Boot Variable content in this region, such as PK, KEK, db, dbx.

The TD HOB list is used to pass the information from VMM to TDVF. The TD HOB must include PHIT HOB, Resource Descriptor HOB. Other HOBs are optional.
- The TD HOB must include PHIT HOB as the first HOB. EfiMemoryTop, EfiMemoryBottom, EfiFreeMemoryTop, and EfiFreeMemoryBottom shall be zero.
- The TD HOB must include at least one Resource Description HOB to declare the physical memory resource.

***0.*** `SecTdxHelperLib` is the SEC instance of  TdxHelperLib. It implements the following
functions for tdx in SEC phase:

- `TdxHelperMeasureTdHob` measure/extend TdHob and store the measurement value in workarea.
- `TdxHelperMeasureCfvImage` measure/extend the Configuration FV image and store the measurement value in workarea.
- `TdxHelperBuildGuidHobForTdxMeasurement` builds GuidHob for TDX measurement.

**Measure Secure Boot Policy to PCR[7]/RTMR[0]**
 
*1.* UEFI Debug Mode.

If a platform provides a firmware debugger mode, then the platform shall measure "UEFI Debug Mode" string with `EV_EFI_ACTION`. This logic is done at TdTcg2Dxe.c `MeasureSecureBootPolicy()`, based upon [`PcdFirmwareDebuggerInitialized`](https://github.com/tianocore/edk2/blob/master/SecurityPkg/SecurityPkg.dec).

*2.* The contents of the `SecureBoot` variable.

*3.* The contents of the `PK` variable.

*4.* The contents of the `KEK` variable.

*5.* The contents of the `EFI_IMAGE_SECURITY_DATABASE` variable (the DB).

*6.* The contents of the `EFI_IMAGE_SECURITY_DATABASE1` variable (the DBX).

*7.* The contents of the `EFI_IMAGE_SECURITY_DATABASE2` (the DBT) variable, if present and not empty.

The UEFI secure boot related variables -- "SecureBoot", "PK", "KEK", "db", and "dbx" are unconditionally measured by TdTcg2Dxe.c `ReadAndMeasureSecureVariable()`. The event type is [`EV_EFI_VARIABLE_DRIVER_CONFIG`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h). If they are not present, a zero size UEFI variable entry will be measured. The "dbt" and "dbr" variables are conditionally measured only if they are present by the routine `MeasureAllSecureVariables()`.

*8.* Separator.

[`EV_SEPARATOR`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h) for PCR7 is handled in TdTcg2Dxe.c `MeasureSecureBootPolicy()` when the UEFI variable is ready. It is just after `MeasureAllSecureVariables()`. It is earlier than the `ReadyToBoot` event signal. The reason is that the PCR7 `EV_SEPARATOR` must be between SecureBootPolicy (Configure) and and ImageVerification (Authority).

**Measure Boot Variable to PCR[1]/RTMR[0]**

*9.* The UEFI BootOrder Variable and the Boot#### variables (just device paths).

The UEFI boot related variables, such as "BootOrder." and "Boot####"  are measured by TdTcg2Dxe.c `ReadAndMeasureBootVariable()`. The event type is [`EV_EFI_VARIABLE_BOOT`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h). These variables are measured if they are present in `MeasureAllBootVariables()`.

**Upon selecting a boot device,**

*10.* The boot attempt action "Calling EFI Application from Boot Option", this means Boot Manager attempting to execute code from a Boot Option (**PCR[4]/RTMR[1]**).

The boot attempt action is measured by TdTcg2Dxe.c `OnReadyToBoot()`. Before invoking a boot option, it measures the action \"Calling EFI Application from Boot Option\". After the boot option returns, it measures the action \"Returning from EFI Application from Boot Option\".

*11.* Separator, Draw a line between leaving pre-boot env and entering post-boot env (**PCR[0~6]/RTMR[1]**).

*12.* **[Optional] If UEFI Secure Boot is enabled,** measure the entry in the `EFI_IMAGE_SECURITY_DATABASE` that was used to validate the UEFI image (**PCR[7]/RTMR[0]**). 

When UEFI secure boot is enabled, the [`DxeImageVerificationLib`](https://github.com/tianocore/edk2/tree/master/SecurityPkg/Library/DxeImageVerificationLib) verifies the PE image signature based upon the [`EFI_SIGNATURE_DATA`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Guid/ImageAuthentication.h) in the [`EFI_SIGNATURE_LIST`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Guid/ImageAuthentication.h) of an image signature database. If an `EFI_SIGNATURE_DATA` is used to verify the image, then this `EFI_SIGNATURE_DATA` will be measured with [`EV_EFI_VARIABLE_AUTHORITY`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h) in `MeasureVariable()` of [Measurement.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/Measurement.c).

*13.* The GUID Partition Table (GPT) disk geometry (**PCR[5]/RTMR[1]**).

When a system boots a boot option in a GUID-named partition of the disk, the GUID partition table (GPT) disk geometry needs to be measured. It is done by [DxeTpm2MeasureBootLib.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeTpm2MeasureBootLib/DxeTpm2MeasureBootLib.c) `Tcg2MeasureGptTable()` in `DxeTpm2MeasureBootHandler()`.

*14.* The selected UEFI application code PE/COFF image, i.e., OS loader (**PCR[4]/RTMR[1]**).

A third party UEFI application, such as a UEFI shell utility, a standard OS loader or an OEM boot option, is measured by DxeTpm2MeasureBootLib.c `Tcg2MeasurePeImage()` in `DxeTpm2MeasureBootHandler()`. The event type is [`EV_EFI_BOOT_SERVICES_APPLICATION`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h). If a UEFI application is an FV which is dispatched in the DXE phase, it is also measured to PCR4 irrespective of whether the FV is measured or unmeasured.

**Then OS loader, i.e., [Grub2](https://git.savannah.gnu.org/cgit/grub.git/) extends the trusted boot chain from virtual firmware into the OS.**

*NOTE: Here we do not concern [Shim](https://github.com/rhboot/shim) component since we only focus on measured boot. Shim is used to extend the UEFI secure boot concept to Linux.*

*15.* Grub2 measures configuration file (e.g., grub.cfg), grub commands, kernel binary, kernel commands and initrd binary (**PCR[8, 9]/RTMR[2]**). PCR[8] is for the command line string and PCR[9] is for a file binary, as shown in the following table.

To support measurements on confidential computing platforms, two patches have been upstreamed, including:
- [efi/tpm: Add EFI_CC_MEASUREMENT_PROTOCOL support](https://git.savannah.gnu.org/cgit/grub.git/commit/?id=4c76565b6cb885b7e144dc27f3612066844e2d19)
- [commands/efi/tpm: Re-enable measurements on confidential computing platforms](https://git.savannah.gnu.org/cgit/grub.git/commit/?id=86df79275d065d87f4de5c97e456973e8b4a649c)

| PCR Index | PCR Usage                                                                                                                                                |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 8         | Grub command line: All executed commands (including those from configuration files) will be logged and measured as entered with a prefix of "grub cmd:" |
|           | Kernel command line: Any command line passed to a kernel will be logged and measured as entered with a prefix of "kernel cmdline:"                      |
|           | Module command line: Any command line passed to a kernel module will be logged and measured as entered with a prefix of "module cmdline:"               |
| 9         | Files: Any file read by GRUB will be logged and measured with a descriptive text corresponding to the filename.                                          |

[tpm.c](https://github.com/rhboot/grub2/blob/master/grub-core/commands/tpm.c) registers `grub_tpm_verify_string()` and `grub_tpm_verify_write()` to a grub_file_verifier structure. They will be called by `grub_verify_string()` and `grub_verifiers_open()` in [verifiers.c](https://github.com/rhboot/grub2/blob/master/grub-core/commands/verifiers.c).

when grub2 executes a command line such as `GRUB_VERIFY_MODULE_CMDLINE`, `GRUB_VERIFY_KERNEL_CMDLINE`, `GRUB_VERIFY_COMMAND` or `grub_create_loader_cmdline()` in [cmdline.c](https://github.com/rhboot/grub2/blob/master/grub-core/lib/cmdline.c), `grub_verify_string()` is used. Finally, `grub_tpm_verify_string()` calls `grub_tpm_measure` and then `grub_cc_log_event` to measure the string to **PCR[8]/RTMR[2]**.

`grub_verifiers_open()` is registered as one of grub_file_filters in [`file.h`](https://github.com/rhboot/grub2/blob/master/include/grub/file.h). Whenever grub uses [file.c](https://github.com/rhboot/grub2/blob/master/grub-core/kern/file.c) `grub_file_open()` this filter is invoked. Finally, `grub_tpm_verify_write()` calls `grub_tpm_measure` and then `grub_cc_log_event` to measure the file binary to **PCR[9]/RTMR[2]**.

*16.* The boot attempt action "Exit Boot Services Invocation", this means Boot Manager has sent the call to UEFI to end Boot Services (**PCR[5]/RTMR[1]**).

*17.* The boot attempt action "Exit Boot Services Returned with Success", this means UEFI successfully existed Boot Services and pre-OS environment has been terminated (**PCR[5]/RTMR[1]**).

The ExitBootServices action is measured by TdTcg2Dxe.c. If ExitBootServices succeeds, then `OnExitBootServices()` is invoked. If ExitBootServices fails, then `OnExitBootServicesFailed()` is invoked.

**If Security Boot Policy update after initial measurement and before `ExitBootServices()` has completed,**

*18.* The platform MAY be restarted OR the variables MUST be remeasured into (**PCR[7]/RTMR[0]**). Additionally the normal update process for setting any of the defined Secure Boot variables SHOULD occur before the initial measurement in PCR[7] or after the call to `ExitBootServices()` has completed.

The UEFI secure boot variable update is measured in Variable [`RuntimeDxe`](https://github.com/tianocore/edk2/tree/master/MdeModulePkg/Universal/Variable/RuntimeDxe). If any of the above secure boot related variables are updated, then [Measurement.c](https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Universal/Variable/RuntimeDxe/Measurement.c) `MeasureVariable()` will measure the new data with `EV_EFI_VARIABLE_DRIVER_CONFIG`.

## Attestation

We will analyse how to verify the TD Quote from the perspective of the verifier, without focusing on the Quote generation or interactions with the attester. At a high level, it can be broken down into Quote Verification, Event Log Replay and Event Parsing.

### Quote Verification

A measured boot root of trust (RoT) can issue a report of the MRs, often called a quote or attestation report. This quote is typically a digitally-signed digest of MRs.

A verifier then checks that the digest of MRs are signed by a trustworthy key, the RoT for Reporting (RTR). This RTR, aka attestation key, is typically a certified key that signs a report of the MRs. This certification is known as an Endorsement in the IETF RATS Architecture. More details about quote verification could be found in [Intel SGX-based Data Center Attestation Primitives (DCAP)](https://github.com/intel/SGXDataCenterAttestationPrimitives).

### Event Log Replay

Event log replay involves deserializing a raw event log and using the events to recalculate all of the measurement registers. Each event contains a digest and a measurement register index. The verifier will create simulated measurement registers and, for each event, extend the event digest into its corresponding simulated register. At the end, the verifier compares the simulated register values against the actual quoted measurement register values from the first step.

Intel provides a Python library called [tdx-tools](https://github.com/canonical/tdx/tree/main/tests/lib/tdx-tools/src/tdxtools) to verify TD measurements. It consists of the following three steps. For the step-by-step flow, please refer to class `VerifyActor` defined in tdx-tools.

* Get TD event log from CCEL ACPI table.
* Get TDREPORT via Linux attestation driver.
* Compare RTMR value from TDREPORT and RTMR value replayed via event log. 

The two values are expected to be identical, which means the measured contents are not tampered with.

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

### Event Parsing

Event parsing is the process of pulling information from the events. The verifier uses the output to make verification decisions against the appraisal policy, endorsements, and reference values. 

In [go-eventlog](https://github.com/google/go-eventlog), `ReplayAndExtract` parses a CC event log and replays the parsed event log against the RTMR bank specified by hash (the second step). It then extracts event info from the verified log into a FirmwareLogState. In this FirmwareLogState, includes details about the firmware, secure boot configuration, bootloaders (such as grub) and kernel (also includes command line and initramfs).

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

### Reference Measurements

The remaining topic is a quite complex challenge: how to generate reference measurements for FirmwareLogState. Now it is still a open issue [Calculate measurements outside of TDX #263](https://github.com/canonical/tdx/issues/263) in the tdx repo. Inspired by a tool called [cvm-image-rewriter](https://github.com/cc-api/cvm-image-rewriter) in cc-api repo, we may design a plugin to calculate reference measurements of the VM image. 

The cvm-image-rewriter tool is plugin-based and used to customize the confidential VM guest including guest image, config, OVMF firmware etc. The following table shows some existing plugins.

| Name | Descriptions |
| ---- | ------------ |
| 01-resize-image | Resize the input qcow2 image |
| 02-motd-welcome | Customize the login welcome message |
| 03-netplan | Customize the netplan.yaml |
| 04-user-authkey | Add auth key for user login instead of password |
| 05-readonly-data | Fix some file permission to ready-only |
| 06-install-tdx-guest-kernel | Install MVP TDX guest kernel |
| 07-device-permission | Fix the permission for device node |
| 08-ccnp-uds-directory-permission | Fix the permission for CCNP UDS directory |
| 60-initrd-update | Update the initrd image |
| 97-sample | plugin customization example |
| 98-ima-enable-simple | Enable IMA (Integrity Measurement Architecture) feature |

After customising the VM image, a plugin (possibly named reference-measurement-calculator) can read the grub and kernel, then compute the digest of the images and configurations. Therefore reference measurements of VM image is obtained for final verification.

Next question is how to get reference measurements about the firmware. According to TDVF Binary Layout, we can parse the virtual firmware, locate the BFV and CFV, and then compute their reference measurements. Here, we recommend referring to this tool named [tdx-measure](https://github.com/edgelesssys/contrast/blob/main/tools/tdx-measure/tdvf/tdvf.go) in contrast repo. 


The tool also attempts another attestation approach. It does RTMR precalulation for given firmware and kernel file (see `newRtMrCmd()` function), and compare RTMR value from TDREPORT and pre-computed RTMR value. This improves verification efficiency by eliminating the need for the event log. It is important to note that this method is suitable for relatively fixed scenarios. The event logs are still necessary in most scenarios, because firmware, bootloaders, OS and applications are free to append measurements for any event, in any order. 

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

## Appendix

<span id="jumpA"></span>

### TDVF Configurations

**Config-A**:
- Merge the *basic* TDVF feature to existing `OvmfX64Pkg.dsc`. (Align with existing SEV)
- Threat model: VMM is NOT out of TCB. (We don't make things worse)
- The `OvmfX64Pkg.dsc` includes SEV/TDX/normal OVMF basic boot capability. The final binary can run on SEV/TDX/normal OVMF.
- No changes to existing OvmfPkgX64 image layout.
- No need to remove features if they exist today.
- PEI phase is NOT skipped in either TD or Non-TD.
- RTMR based measurement is supported.
- External inputs from Host VMM are measured, such as TdHob, CFV.
- Other external inputs are measured, such as FW_CFG data, os loader, initrd, etc.

**Build the TDVF (Config-A) target:**

[OvmfPkg: Support Tdx measurement in OvmfPkgX64](https://git.codelinaro.org/linaro/dcap/edk2/-/commit/4d37059d8e1eeda124270a158416795605327cbd)

```shell
cd /path/to/edk2
source edksetup.sh
build.sh -p OvmfPkg/OvmfPkgX64.dsc -a X64 -t GCC5
```

 **Config-B**:
- Add a standalone `IntelTdx.dsc` to a TDX specific directory for a *full*  feature TDVF.(Align with existing SEV)
- Threat model: VMM is out of TCB. (We need necessary change to prevent attack from VMM)
- `IntelTdx.dsc` includes TDX/normal OVMF basic boot capability. The final binary can run on TDX/normal OVMF.
- It might eventually merge with AmdSev.dsc, but NOT at this point of time. And we don't know when it will happen. We need sync with AMD in the community after both of us think the solutions are mature to merge.
- Need to add necessary security feature as mandatory requirement, such as RTMR based Trusted Boot support.
- Need to measure the external input from Host VMM, such as TdHob, CFV.
- Need to measure other external input, such as FW_CFG data, os loader, initrd, etc.
- Need to remove unnecessary attack surfaces, such as network stack.

**Build the TDVF (Config-B) target:**

[OvmfPkg: Introduce IntelTdxX64 for TDVF Config-B](https://git.codelinaro.org/linaro/dcap/edk2/-/commit/44a53a3bdd9c76e37f1750b5aa6a745de5d77391)

```shell
cd /path/to/edk2
set PACKAGES_PATH=/path/to/edk2/OvmfPkg
source edksetup.sh
build.sh -p OvmfPkg/IntelTdx/IntelTdxX64.dsc -a X64 -t GCC5
```

<span id="jumpB"></span>

### UEFI Boot Phase

UEFI has six main boot phases, which are all critical in the initialization process of the platform. The combined phases are referred to as the Platform Initialization or PI.

*1.* Security (SEC)

This phase is the primary stage of the UEFI boot process, and will generally be used to: initialize a temporary memory store, act as the root of trust in the system and provide information to the Pre-EFI core phase. This root of trust is a mechanism that ensures any code that is executed in the PI is cryptographically validated (digitally signed), creating a “secure boot” environment.

*2.* Pre-EFI Initialization (PEI)

This is the second stage of the boot process and involves using only the CPU’s current resources to dispatch Pre-EFI Initialization Modules (PEIMs). These are used to perform initialization of specific boot-critical operations such as memory initialization, whilst also allowing control to pass to the Driver Execution Environment (DXE).

*3.* Driver Execution Environment (DXE)

The DXE phase is where the majority of the system initialization occurs. In the PEI stage, the memory required for the DXE to operate is allocated and initialized, and upon control being passed to the DXE, the DXE Dispatcher is then invoked. The dispatcher will perform the loading and execution of hardware drivers, runtime services, and any boot services required for the operating system to start.

*4.* Boot Device Selection (BDS)

Upon completion of the DXE Dispatcher executing all DXE drivers, control is passed to the BDS. This stage is responsible for initializing console devices and any remaining devices that are required. The selected boot entry (OS loader) is then loaded and executed in preparation for the Transient System Load (TSL).

*5.* Transient System Load (TSL)

In this phase, the PI process is now directly between the boot selection and the expected hand-off to the main operating system phase. Here, an application such as the UEFI shell may be invoked, or (more commonly) a boot loader will run in order to prepare the final OS environment. The boot loader is usually responsible for terminating the UEFI Boot Services via the ExitBootServices() call. However, it is also possible for the OS itself to do this, such as the Linux kernel with CONFIG_EFI_STUB.

*6.* Runtime (RT)

The final phase is the runtime one. Here is where the final handoff to the OS occurs. The UEFI compatible OS now takes over the system. The UEFI runtime services remain available for the OS to use, such as for querying and writing variables from NVRAM.