---
layout: post
title: "Intel TDX: Measured Boot and Remote Attestation with GRUB"
date: 2025-01-12
description: How a GRUB-booted TD is measured from firmware to kernel, and how a verifier evaluates that boot chain.
tags: security
giscus_comments: true
# related_posts: true
toc: true
---

Intel Trust Domain Extensions (TDX) places a virtual machine in a hardware-isolated Trust Domain (TD). The TD's memory and execution state are not directly exposed to the virtual machine manager (VMM), hypervisor, or other non-TD software on the host. The cloud service provider still launches and schedules the VM, but control of the host software stack no longer puts that software in the TD's trusted computing base (TCB) by default.

Isolation answers whether the host can directly inspect or modify a TD. It does not answer what the TD booted. Measured boot and remote attestation address the second question together:

1. Firmware, boot configuration, GRUB, the kernel, and the initrd are hashed as they are executed or loaded.
2. Each digest is extended, in order, into the Trust Domain Measurement Register (MRTD) or a Runtime Measurement Register (RTMR), and a corresponding event is logged.
3. A TDX Quote provides cryptographic evidence for TD state that includes these measurements.
4. A verifier validates the Quote, replays the event log, and decides whether the observed boot is acceptable under its reference values and policy.

The UEFI environment inside a TD is commonly provided by TDX Virtual Firmware (TDVF), an EDK2/OVMF-based firmware that establishes the TD firmware environment and loads later components. This article is a mechanism explainer for the GRUB boot path: first, how TDVF hands the measurement chain to GRUB; then, how a verifier connects the Quote, event log, and reference values. The TCB ultimately depends on verification policy. The CPU and TDX Module/SEAM firmware form the platform foundation, while trust in guest components such as TDVF, GRUB, and the kernel depends on the reference values accepted by the verifier. The two TDVF configurations and their different threat models are described in the [appendix](#jumpA).

## TD Boot Types

Start with the two common boot paths shown below.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/td_guest_boot_process.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 1. TD Guest Boot Process
</div>

**Direct boot** has no intermediate bootloader such as GRUB. Firmware obtains the kernel, initrd, and command line through QEMU's `fw_cfg` interface and starts the kernel directly.

**GRUB boot** has firmware load a UEFI bootloader from a disk image. GRUB then reads its configuration, selects a kernel, and loads the initrd. This path introduces another software and configuration layer, so it requires a longer measurement chain.

Figure 2 expands both paths. This article covers only the GRUB path on the right.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/detailed_flow_for_different_td_boot.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. Detailed Flow for Different TD Boot
</div>

The chain has three handoff points. TDVF measures its configuration and external inputs. The UEFI image-loading path measures PE/COFF images such as shim and GRUB. GRUB then measures its configuration and commands, the kernel, and the initrd. When UEFI Secure Boot is enabled, shim can also extend the distribution's authorization chain to GRUB and the kernel. Secure Boot and measured boot serve different purposes: Secure Boot blocks unauthorized code, while measured boot records what was actually loaded.

The event types and PCR semantics come from the TCG measured-boot model. TDX maps those events to MRTD and the RTMRs and includes hardware and TDX Module state in the platform evidence protected by the Quote. The TDVF configuration still determines whether the VMM is actually outside the TCB; using TDX alone is not enough to make that claim.

## UEFI Boot Sequence

GRUB runs as a UEFI application on this path. Figure 3 shows the UEFI boot sequence and the boundary between firmware and the loader.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/uefi_booting_sequence.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 3. UEFI Booting Sequence
</div>

Firmware uses Boot Services (BS) to load and start the selected OS loader. GRUB and other loaders run as UEFI applications and call those services. Once the kernel is ready, the loader calls `ExitBootServices()` to terminate BS and release their resources; only UEFI Runtime Services (RT) remain available to the OS. `ExitBootServices()` is therefore both an execution boundary and a key event-log transition between the pre-boot and post-boot environments.

A detailed introduction to UEFI boot phases refers to [Appendix: UEFI Boot Phase](#jumpB) from [UEFI boot phases](https://secret.club/2020/05/26/introduction-to-uefi-part-1.html#uefi-boot-phases).

## Measured Boot Component

As Figure 4 shows, the pre-boot environment consists of the TDVF/OVMF phase and the bootloader phase, which may include shim and GRUB. UEFI components use `EFI_CC_MEASUREMENT_PROTOCOL` to log events and extend their digests into Runtime Measurement Registers (RTMRs). MRTD records the initial construction measurement of the TD.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/td_measurement_process.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 4. TD Measurement Process
</div>

`EFI_CC_MEASUREMENT_PROTOCOL` follows the TCG event-log model. The Confidential Computing Event Log (CCEL) ACPI table describes the memory region containing the events, and each event digest is extended as `MR_new = Hash(MR_old || event_digest)`. The event log is not itself a root of trust. A verifier must replay it and compare the result with the RTMR values protected by the Quote to detect missing or modified events.

The Measured Boot Component in EDK2 is as follows.

{% include figure.liquid path="assets/img/2025-01-12-tdx-measure-boot/measured_boot_component_in_edk2.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 5. Measured Boot Component in EDK2
</div>

[`SecTdxHelperLib`](https://github.com/tianocore/edk2/tree/master/OvmfPkg/IntelTdx/TdxHelperLib) provides measurement functions during SEC.
[`EFI_CC_MEASUREMENT_PROTOCOL`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Protocol/CcMeasurement.h) abstracts confidential-computing measurement operations for UEFI guests.
[TdTcg2Dxe.c](https://github.com/tianocore/edk2/blob/master/OvmfPkg/Tcg/TdTcg2Dxe/TdTcg2Dxe.c) implements the DXE driver responsible for measurements during DXE.
[`DxeTpm2MeasureBootLib`](https://github.com/tianocore/edk2/tree/master/SecurityPkg/Library/DxeTpm2MeasureBootLib) handles PE image and GPT measurements. [`UefiTcgPlatform.h`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h) defines the event types used below.

The following interface is defined by the [Confidential Computing chapter of the UEFI Specification](https://uefi.org/specs/UEFI/2.10/38_Confidential_Computing.html#).

CC-capable virtual firmware that supports measurement exposes `EFI_CC_MEASUREMENT_PROTOCOL` under `EFI_CC_MEASUREMENT_PROTOCOL_GUID`. The protocol reports the event log and provides hashing and extension operations.

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

The interface has four operations:

- `GetCapability` returns protocol capability and state information.
- `GetEventLog` returns the location of an event log and its last entry.
- `HashLogExtendEvent` extends a measurement and optionally logs the event without exposing a technology-specific CC command.
- `MapPcrToMrIndex` maps a TPM PCR index to the corresponding CC measurement-register index.

### Intel TDX Register Mapping

TDX has fewer measurement registers than the TPM PCR model whose event semantics it reuses. The typical mapping is:

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

_Note: RTMR[3] is reserved for specialized uses such as a virtual TPM. It remains available to the guest when no such use claims it._

The registers are normally divided as follows; see [8.1 Measurement Register Usage in TD](https://cdrdv2.intel.com/v1/dl/getContent/733585) for details.

- MRTD measures TDVF code, corresponding to PCR[0].
- RTMR[0] measures TDVF configuration, corresponding to PCR[1] and PCR[7], following the TCG Platform Firmware Profile (PFP).
- RTMR[1] measures components loaded by TDVF, such as the OS loader, corresponding to PCR[4] and PCR[5], also following the TCG PFP.
- RTMR[2] measures OS components such as the kernel, initrd, and applications, corresponding to PCR[8] through PCR[15]. Its use is defined by the OS.
- RTMR[3] is reserved for specialized uses.

## Measured Boot Flow

The following sections walk through the same path as Figure 4 in event order. TDVF first handles VMM-supplied inputs, UEFI then measures boot policy and the OS loader, and GRUB extends the chain to the kernel and initrd. Steps 0 through 18 correspond to the event nodes in the figure.

### Measuring the TD HOB and Configuration FV

[4.2 TD Hand-Off Block (HOB)](https://cdrdv2.intel.com/v1/dl/getContent/733585) and [3.2 Configuration Firmware Volume (CFV)](https://cdrdv2.intel.com/v1/dl/getContent/733585) are supplied by the host VMM, so the TD guest cannot trust them by default. TDVF validates their data structures, measures them, and extends the results into RTMRs. Measurement makes later modification or disagreement visible; it does not turn untrusted input into trusted input. TDVF also creates two `EFI_CC_EVENT_HOB` structures carrying the TD HOB list and CFV digests, which the DXE phase uses to create `EFI_CC_EVENT` records.

The Configuration Firmware Volume contains provisioned, read-only data. It may carry UEFI Secure Boot variables such as PK, KEK, db, and dbx.

The VMM uses the TD HOB list to pass information to TDVF. It must contain a PHIT HOB and at least one Resource Descriptor HOB; other HOBs are optional.

- The TD HOB must include PHIT HOB as the first HOB. EfiMemoryTop, EfiMemoryBottom, EfiFreeMemoryTop, and EfiFreeMemoryBottom shall be zero.
- The TD HOB must include at least one Resource Description HOB to declare the physical memory resource.

**_0._** `SecTdxHelperLib` is the SEC instance of `TdxHelperLib`. It provides three TDX measurement functions during SEC:

- `TdxHelperMeasureTdHob` measures and extends the TD HOB, then stores the result in the work area.
- `TdxHelperMeasureCfvImage` measures and extends the Configuration FV, then stores the result in the work area.
- `TdxHelperBuildGuidHobForTdxMeasurement` builds the GUID HOBs used to carry TDX measurements.

### Measuring Secure Boot Policy into PCR[7]/RTMR[0]

_1._ UEFI Debug Mode.

If a platform provides a firmware debugger mode, it measures the string "UEFI Debug Mode" as `EV_EFI_ACTION`. `MeasureSecureBootPolicy()` in `TdTcg2Dxe.c` implements this rule based on [`PcdFirmwareDebuggerInitialized`](https://github.com/tianocore/edk2/blob/master/SecurityPkg/SecurityPkg.dec).

_2._ The contents of the `SecureBoot` variable.

_3._ The contents of the `PK` variable.

_4._ The contents of the `KEK` variable.

_5._ The contents of the `EFI_IMAGE_SECURITY_DATABASE` variable (the DB).

_6._ The contents of the `EFI_IMAGE_SECURITY_DATABASE1` variable (the DBX).

_7._ The contents of the `EFI_IMAGE_SECURITY_DATABASE2` (the DBT) variable, if present and not empty.

`ReadAndMeasureSecureVariable()` in `TdTcg2Dxe.c` always measures `SecureBoot`, `PK`, `KEK`, `db`, and `dbx` as [`EV_EFI_VARIABLE_DRIVER_CONFIG`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h). If a variable is absent, it records a zero-length UEFI variable entry. `MeasureAllSecureVariables()` measures `dbt` and `dbr` only when they are present.

_8._ Separator.

When UEFI variables are ready, `MeasureSecureBootPolicy()` records the PCR[7] [`EV_SEPARATOR`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h) after `MeasureAllSecureVariables()` and before the `ReadyToBoot` signal. This places the separator between SecureBootPolicy (Configuration) and ImageVerification (Authority).

### Measuring Boot Variables into PCR[1]/RTMR[0]

_9._ The UEFI BootOrder Variable and the Boot#### variables (just device paths).

`ReadAndMeasureBootVariable()` measures UEFI boot variables such as `BootOrder` and `Boot####` as [`EV_EFI_VARIABLE_BOOT`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h). `MeasureAllBootVariables()` records them when present.

### After Selecting a Boot Device

_10._ The action "Calling EFI Application from Boot Option" records that Boot Manager is about to execute code from a boot option (**PCR[4]/RTMR[1]**).

`OnReadyToBoot()` in `TdTcg2Dxe.c` records the action "Calling EFI Application from Boot Option" before invoking a boot option. If that option returns, the next boot attempt records "Returning from EFI Application from Boot Option" before trying another option.

_11._ A separator marks the transition from the pre-boot environment to the post-boot environment. In the TCG model, separator events are written to PCR[0] through PCR[6]. Here the RTMR[0] separator was already extended after Secure Boot Policy measurement, so `OnReadyToBoot()` extends the remaining separator into **RTMR[1]**.

_12._ **[Optional] If UEFI Secure Boot is enabled,** measure the entry in the `EFI_IMAGE_SECURITY_DATABASE` that was used to validate the UEFI image (**PCR[7]/RTMR[0]**).

When UEFI Secure Boot is enabled, [`DxeImageVerificationLib`](https://github.com/tianocore/edk2/tree/master/SecurityPkg/Library/DxeImageVerificationLib) verifies the PE image signature against an [`EFI_SIGNATURE_DATA`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Guid/ImageAuthentication.h) entry in an [`EFI_SIGNATURE_LIST`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Guid/ImageAuthentication.h). If an entry authorizes the image, `MeasureVariable()` in [Measurement.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/Measurement.c) measures that entry as [`EV_EFI_VARIABLE_AUTHORITY`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h).

_13._ The GUID Partition Table (GPT) disk geometry (**PCR[5]/RTMR[1]**).

When the selected boot option is on a GPT partition, `DxeTpm2MeasureBootHandler()` calls `Tcg2MeasureGptTable()` in [DxeTpm2MeasureBootLib.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeTpm2MeasureBootLib/DxeTpm2MeasureBootLib.c) to measure the disk layout.

_14._ The selected UEFI application's PE/COFF image—that is, the OS loader (**PCR[4]/RTMR[1]**).

`DxeTpm2MeasureBootHandler()` uses `Tcg2MeasurePeImage()` to measure third-party UEFI applications such as shell utilities, standard OS loaders, and OEM boot options as [`EV_EFI_BOOT_SERVICES_APPLICATION`](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/IndustryStandard/UefiTcgPlatform.h). A UEFI application dispatched from an FV during DXE is measured into PCR[4] regardless of whether the containing FV was measured separately.

### GRUB Extends the Measurement Chain into the OS

[GRUB 2](https://github.com/rhboot/grub2) then extends the measurement chain from virtual firmware into the OS.

_Note: [shim](https://github.com/rhboot/shim) primarily extends the UEFI Secure Boot authorization chain into the Linux ecosystem. This article does not explain shim's verification logic, but when Secure Boot is enabled, shim itself and its related authority events remain part of the measured boot chain._

_15._ GRUB 2 measures its configuration file (for example, `grub.cfg`), GRUB commands, the kernel binary and command line, and the initrd binary (**PCR[8, 9]/RTMR[2]**). PCR[8] records command strings, while PCR[9] records file contents, as shown below.

To support measurements on confidential computing platforms, two patches have been upstreamed, including:

- [efi/tpm: Add EFI_CC_MEASUREMENT_PROTOCOL support](https://github.com/rhboot/grub2/commit/4c76565b6cb885b7e144dc27f3612066844e2d19)
- [commands/efi/tpm: Re-enable measurements on confidential computing platforms](https://github.com/rhboot/grub2/commit/86df79275d065d87f4de5c97e456973e8b4a649c)

| PCR Index | PCR Usage                                                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 8         | Grub command line: All executed commands (including those from configuration files) will be logged and measured as entered with a prefix of "grub cmd:" |
|           | Kernel command line: Any command line passed to a kernel will be logged and measured as entered with a prefix of "kernel cmdline:"                      |
|           | Module command line: Any command line passed to a kernel module will be logged and measured as entered with a prefix of "module cmdline:"               |
| 9         | Files: Any file read by GRUB will be logged and measured with a descriptive text corresponding to the filename.                                         |

[tpm.c](https://github.com/rhboot/grub2/blob/master/grub-core/commands/tpm.c) registers `grub_tpm_verify_string()` and `grub_tpm_verify_write()` as a `grub_file_verifier`. They are called by `grub_verify_string()` and `grub_verifiers_open()` in [verifiers.c](https://github.com/rhboot/grub2/blob/master/grub-core/kern/verifiers.c).

When GRUB 2 handles `GRUB_VERIFY_MODULE_CMDLINE`, `GRUB_VERIFY_KERNEL_CMDLINE`, or `GRUB_VERIFY_COMMAND`, or calls `grub_create_loader_cmdline()` in [cmdline.c](https://github.com/rhboot/grub2/blob/master/grub-core/lib/cmdline.c), it invokes `grub_verify_string()`. `grub_tpm_verify_string()` then calls `grub_tpm_measure()`, and `grub_cc_log_event()` measures the string into **PCR[8]/RTMR[2]**.

`grub_verifiers_open()` is registered as one of the `grub_file_filter` entries in [`file.h`](https://github.com/rhboot/grub2/blob/master/include/grub/file.h). Whenever GRUB calls `grub_file_open()` in [file.c](https://github.com/rhboot/grub2/blob/master/grub-core/kern/file.c), the filter runs. `grub_tpm_verify_write()` then calls `grub_tpm_measure()`, and `grub_cc_log_event()` measures the file contents into **PCR[9]/RTMR[2]**.

_16._ "Exit Boot Services Invocation" records that Boot Manager has called UEFI to end Boot Services (**PCR[5]/RTMR[1]**).

_17._ "Exit Boot Services Returned with Success" records that UEFI successfully exited Boot Services and terminated the pre-OS environment (**PCR[5]/RTMR[1]**).

`TdTcg2Dxe.c` records the `ExitBootServices()` transition through `OnExitBootServices()` on success or `OnExitBootServicesFailed()` on failure.

### If Secure Boot Policy Changes During Boot

_18._ The platform MAY be restarted OR the variables MUST be remeasured into (**PCR[7]/RTMR[0]**). Additionally the normal update process for setting any of the defined Secure Boot variables SHOULD occur before the initial measurement in PCR[7] or after the call to `ExitBootServices()` has completed.

UEFI Secure Boot variable updates are measured by [`RuntimeDxe`](https://github.com/tianocore/edk2/tree/master/MdeModulePkg/Universal/Variable/RuntimeDxe). When one of the variables above changes, `MeasureVariable()` in [Measurement.c](https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Universal/Variable/RuntimeDxe/Measurement.c) records the new data as `EV_EFI_VARIABLE_DRIVER_CONFIG`.

## Attestation

From a verifier's perspective, remote attestation is not a single check that asks whether the Quote signature is valid. It is a three-stage pipeline: establish the Quote's provenance and integrity, use its MRTD and RTMR values to validate the event log, then interpret the verified events as firmware, Secure Boot, GRUB, and kernel state and compare that state with policy and reference values. Quote generation and transport are outside this article's scope.

### Quote Verification

A TDX Quote carries the contents of TDREPORT together with signatures and certification material needed for verification. A verifier checks the signature, certificate chain, revocation and TCB status, and freshness needed to prevent replay. It must also confirm that the reported platform and TD attributes satisfy policy. Successfully parsing a Quote does not by itself establish that the workload is trustworthy. The attestation key and its certification material form part of the endorsements in the IETF RATS architecture. See [Intel SGX Data Center Attestation Primitives (DCAP)](https://github.com/intel/SGXDataCenterAttestationPrimitives) for the concrete formats and verification flow.

### Event Log Replay

Event log replay involves deserializing a raw event log and using the events to recalculate all of the measurement registers. Each event contains a digest and a measurement register index. The verifier will create simulated measurement registers and, for each event, extend the event digest into its corresponding simulated register. At the end, the verifier compares the simulated register values against the actual quoted measurement register values from the first step.

Canonical's TDX test repository contains the Python tool [tdx-tools](https://github.com/canonical/tdx/tree/main/tests/lib/tdx-tools/src/tdxtools), which compares event-log replay results with the RTMR values in TDREPORT from inside a TD guest. Its `VerifyActor` illustrates three steps:

- Get TD event log from CCEL ACPI table.
- Get TDREPORT via Linux attestation driver.
- Compare RTMR value from TDREPORT and RTMR value replayed via event log.

An exact match establishes that the event log and the RTMR values in TDREPORT are consistent. It does not establish that the measured components satisfy the verifier's policy; that decision requires event parsing and reference values. The following excerpt shows the current implementation's core path:

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

### Event Parsing

Event parsing is the process of pulling information from the events. The verifier uses the output to make verification decisions against the appraisal policy, endorsements, and reference values.

[go-eventlog](https://github.com/google/go-eventlog/blob/main/extract/extract.go), for example, can parse, replay, and extract state from several measured-boot log formats, including firmware, Secure Boot configuration, GRUB, and Linux kernel state. The library explicitly does not verify Quotes or provide reference measurements. Integrating code must verify the Quote first and then supply trusted register values to the replay and extraction logic. The `FirmwareLogState` excerpt below shows this extraction step; the exact API may continue to change while the library remains pre-release.

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

### Reference Measurements

After establishing log consistency, the verifier must still ask whether the components are the expected versions. That requires reference measurements or a higher-level Reference Integrity Manifest (RIM). The final register values depend not only on file contents, but also on loaded representations, dynamic platform data, and event order. Hashing a few files independently with SHA-384 is therefore not a complete reference-value calculation.

Canonical issue [#263](https://github.com/canonical/tdx/issues/263) documents these difficulties in offline MRTD and RTMR calculation. It was closed in July 2026 when the original Tech Preview was archived, not because it produced a general algorithm for arbitrary GRUB boot configurations. The following is an author proposal, not a capability guaranteed by an existing tool: inspired by the plugin architecture of [cvm-image-rewriter](https://github.com/cc-api/cvm-image-rewriter), an image build could add a `reference-measurement-calculator` that extracts GRUB, the kernel, the initrd, and configuration and emits reference material bound to that build.

`cvm-image-rewriter` uses plugins to customize a confidential VM's guest image, configuration, and OVMF firmware. The repository includes the following plugins:

| Name                             | Descriptions                                            |
| -------------------------------- | ------------------------------------------------------- |
| 01-resize-image                  | Resize the input qcow2 image                            |
| 02-motd-welcome                  | Customize the login welcome message                     |
| 03-netplan                       | Customize the netplan.yaml                              |
| 04-user-authkey                  | Add auth key for user login instead of password         |
| 05-readonly-data                 | Fix some file permissions to read-only                  |
| 06-install-tdx-guest-kernel      | Install MVP TDX guest kernel                            |
| 07-device-permission             | Fix the permission for device node                      |
| 08-ccnp-uds-directory-permission | Fix the permission for CCNP UDS directory               |
| 60-initrd-update                 | Update the initrd image                                 |
| 97-sample                        | plugin customization example                            |
| 98-ima-enable-simple             | Enable IMA (Integrity Measurement Architecture) feature |

Such a plugin would need to record component digests, event types, destination registers, and extension order. If the measured object is an in-memory representation rather than the original file, the plugin must also reproduce the relevant loading logic. Its output should be treated as reference material for a specific build and boot configuration, not as an RTMR value that applies to every platform.

On the firmware side, TDVF Binary Layout can be used to locate the BFV and CFV in a firmware image. Contrast's [tdx-measure](https://github.com/edgelesssys/contrast/blob/main/tools/tdx-measure/tdvf/tdvf.go) demonstrates how to precompute selected measurements for explicit inputs, but its assumptions must match the actual TDVF build, QEMU parameters, and boot path.

The tool also exposes an RTMR precomputation command for a given firmware, kernel, initrd, and command line through `newRtMrCmd()`. When inputs and event order are completely fixed, a verifier can compare a precomputed value directly with the reported value. Dynamic ACPI data, boot-menu choices, or additional events can invalidate that comparison. The event log therefore remains essential evidence for explaining which object affected a register and in what order.

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

- Merge the _basic_ TDVF feature into the existing `OvmfPkgX64.dsc`. (Align with existing SEV.)
- Threat model: the VMM remains in the TCB.
- `OvmfPkgX64.dsc` provides basic boot support for SEV, TDX, and normal OVMF. The resulting binary can run in all three environments.
- The existing OvmfPkgX64 image layout is unchanged.
- Existing features do not have to be removed.
- PEI is not skipped for either TD or non-TD guests.
- RTMR-based measurement is optional and can be enabled.
- Host VMM inputs such as the TD HOB and CFV are measured.
- Other external inputs, including `fw_cfg` data, the OS loader, and the initrd, are measured.

**Build TDVF Config-A:**

```shell
cd /path/to/edk2
. ./edksetup.sh
make -C BaseTools
build -p OvmfPkg/OvmfPkgX64.dsc -a X64 -t GCC \
  -D CC_MEASUREMENT_ENABLE=TRUE -b RELEASE
```

**Config-B**:

- Add a standalone `IntelTdxX64.dsc` under the TDX-specific directory for a full-featured TDVF. (Align with existing SEV.)
- Threat model: the VMM is outside the TCB, so the firmware must defend against VMM-controlled inputs.
- `IntelTdxX64.dsc` provides basic boot support for TDX and normal OVMF. The resulting binary can run in both environments.
- It may eventually merge with `AmdSev.dsc`, but the projects have not committed to a schedule.
- RTMR-based measurement is mandatory.
- Host VMM inputs such as the TD HOB and CFV are measured.
- Other external inputs, including `fw_cfg` data, the OS loader, and the initrd, are measured.
- Skip PEI and split the DXE FV so a TD guest loads only the required drivers, reducing the attack surface.

**Build TDVF Config-B:**

```shell
cd /path/to/edk2
. ./edksetup.sh
make -C BaseTools
build -p OvmfPkg/IntelTdx/IntelTdxX64.dsc -a X64 -t GCC -b RELEASE
```

These configuration notes and commands follow the EDK2 [TDVF README](https://github.com/tianocore/edk2/blob/master/OvmfPkg/IntelTdx/README.md). Always use the README from the EDK2 revision being built as the final authority.

<span id="jumpB"></span>

### UEFI Boot Phase

This article uses the common six-phase view of UEFI platform initialization (PI): SEC, PEI, DXE, BDS, TSL, and Runtime.

_1._ Security (SEC)

This is the entry point to PI. It establishes the initial execution environment and temporary memory, then passes platform boot information to PEI. SEC may host a platform root of trust, but that is not the same as UEFI Secure Boot, whose signature policy is primarily enforced later when images are loaded.

_2._ Pre-EFI Initialization (PEI)

PEI uses the CPU resources available at that point to dispatch Pre-EFI Initialization Modules (PEIMs). These modules perform boot-critical work such as memory initialization, then pass control to the Driver Execution Environment (DXE).

_3._ Driver Execution Environment (DXE)

DXE performs most system initialization. After PEI has prepared memory and transferred control, the DXE Dispatcher loads hardware drivers, Runtime Services, and the Boot Services needed to start the OS.

_4._ Boot Device Selection (BDS)

After the DXE Dispatcher has run the required drivers, Boot Device Selection initializes consoles and any remaining required devices, selects a boot entry, and loads its OS loader in preparation for TSL.

_5._ Transient System Load (TSL)

TSL spans the interval between boot selection and the final OS handoff. A UEFI application such as the shell may run here, but the usual path executes a bootloader that prepares the OS and calls `ExitBootServices()`. An OS can also make that call directly, as a Linux kernel built with `CONFIG_EFI_STUB` does.

_6._ Runtime (RT)

The OS takes control during Runtime. UEFI Runtime Services remain available, including services for reading and writing variables in NVRAM.
