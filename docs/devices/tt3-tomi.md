# TT3 “Tomi”

## Scope and authority

This profile records durable TT3 device identity and platform facts, separating them from changeable software and operating state. The live device name in project use is **Tomi**; its captured TomTom identity is the **TomTom XXL IQ Routes Edition**, identified in `ttgo.bif` as hardware version **ONE XXL IQ Routes**.

The strongest local evidence is Tomi's own `ttgo.bif`, bootloader version file, and captured Linux boot log. Local evidence paths and confidence limits are summarized in the external reconciliation record. Per-unit serial and unique-ID fields in `ttgo.bif` are intentionally not copied into this repository.

## Identity

| Field | Established value | Evidence boundary |
|---|---|---|
| Project identifier | `TT3` | Project convention. |
| Informal name | `Tomi` | Project convention. |
| TomTom device name | `TomTom XXL IQ Routes Edition` | `ttgo.bif` and Linux device-detection log agree. |
| Hardware-version string | `ONE XXL IQ Routes` | Device-generated `ttgo.bif`. |
| Regulatory model / product number | `N14644` / `4EP0.001.04` | TT3 archival runbook; original label/intake image was not present in the inspected local corpus. |
| Linux hardware-detection type | Type `42`, detected name `TomTom XXL IQ Routes Edition` | Tomi kernel boot log. OpenTom names enum value 42 `GOTYPE_AUSTIN`; this is a software platform-profile name, not a different retail model. |
| Bootloader | `s3c24xx`, version `5.5279` | Captured `bootloaderversion.txt`; `ttgo.bif` independently reports numeric `55279`. |

Unique per-device identifiers exist in the preserved intake data. They are omitted here because this canonical file is published through GitHub; consult the private source evidence when the unique ID is operationally required.

## Hardware profile

### Processor and memory

Tomi's captured kernel log reports an ARM926EJ-S core (`ARMv5TEJ`) and Samsung S3C2412 (`id 0x32412000`), with 64 MB total RAM. The same boot record reports core/memory/peripheral clocks of 264/132/66 MHz. The kernel also enables TomTom CPU-frequency scaling, so those boot-reported clock values should not be treated as a guaranteed fixed runtime frequency.

The corresponding OpenTom type-42/Austin profile declares S3C2412. Do not generalize this identity from another TomTom model.

### Display and input

The TT3 archival identification record reports LCD `AUO A050FW03V2 ALT 28`. The OpenTom Austin profile instead selects the `GOTFT_AUO_A050FW02V2` software enum for type 42. The original raw display-diagnostic output is not present in the inspected corpus, and the enum/string difference is not resolved here; preserve the reported physical display identity with that provenance caveat rather than assuming the two labels are equivalent.

Tomi's captured kernel identifies the S3C2410 LCD controller and TomTom framebuffer driver. It also reports the TomTom touchpad driver. Exact panel resolution, touch-controller part number, and board revision are not established by the cited evidence.

### GPS

The type-42 Austin software profile selects `GOGPS_GL_BCM4750` and marks GL GPS detected. The captured `ttgo.bif` reports GPS firmware version `2.15.0 65947`. These are platform/software identifiers; the physical receiver package and whether that firmware string remains current are not independently established. The active GPS stack and application versions are mutable software state and are not specified as device identity here.

### Storage

Tomi's Linux boot log identifies the main block device as `mmcblk0`, with MMC product string `M4G1EM`, reported capacity `3907584 KiB`, and partition `p1`. The 2026-09-24 live-state capture reports `/dev/mmcblk0p1` mounted as VFAT at `/mnt/sdcard`; `/mnt/sdcard/opentom` is the current OpenTom root. These observations establish the Linux-visible storage path and reported device identity, not the physical package construction or a complete board storage specification.

The Linux-visible flash filesystem is NGFFS on `mtd0`, mounted at `/mnt/flash` in the current live-state capture. The historical captured `ttsystem` also describes NGFFS and main FAT storage, but its device-node and mount setup is software-version-specific. Do not infer physical flash capacity from the filesystem mount alone.

## USB and platform interfaces

The Tomi kernel exposes the platform role interface:

```text
/sys/devices/platform/tomtomgo-usbmode/mode
```

OpenTom's type-42/Austin profile identifies S3C24xx USB and one OHCI host port. The current project setup uses Tomi as the USB-host side of the TomiDock CDC-ECM connection. The interface is a platform characteristic; its reported role is mutable state, not a permanent property of the device.

The captured kernel also includes S3C24xx UART, RTC, framebuffer, MMC/SDI, USB host/device-mode, and TomTom power/GPS platform drivers. This is an inventory of observed kernel interfaces, not a guarantee that every controller is externally accessible or enabled in every software state.

## Software baseline and historical separation

The current live-state capture supplied for this migration reports:

```text
Linux (none) 2.6.13 #9 Wed Sep 9 13:11:52 UTC 2026 armv5tejl GNU/Linux
```

Treat this as a dated live software observation, not a hardware identity. The supplied 2026-09-24 live-state notes additionally report the mount layout above and `USB_STATE_HOST`; that role value can change.

Two preserved `ttsystem` specimens must not be conflated:

- The 2026-08-04 filesystem capture contains a 4,212,715-byte `ttsystem`, SHA-256 `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf`. Its embedded kernel identifies itself as `Linux 2.6.13-tt567329`, build `#1`, dated 2010-08-24. This is a historical image specimen, not the current live kernel.
- The preserved live-baseline file acquired 2026-08-31 is 2,219,582 bytes, SHA-256 `14505f7b17f1037163ea448ec418924d1302f575db3a7c65ef416a42af4efc21`. The 2026-09-24 live kernel line is supplied separately; the local evidence does not cryptographically bind that exact capture line to the August file specimen.

Current runtime command compatibility belongs in [Tomi Runtime ABI](../reference/tomi-runtime-abi.md). File movement belongs in [Tomi File Transfer](../runbooks/tomi-file-transfer.md). Detailed TomiDock topology and design decisions belong in its architecture document, not this device profile.

## Durable filesystem conventions

The live OpenTom tree is rooted at `/mnt/sdcard/opentom`, on the main VFAT storage mounted at `/mnt/sdcard`. NGFFS is exposed at `/mnt/flash`. These are useful TT3 path conventions, but installed contents and mount implementation can change with software.

## Canonical references

- [Tomi Runtime ABI](../reference/tomi-runtime-abi.md) — current command/tool compatibility.
- [Tomi File Transfer](../runbooks/tomi-file-transfer.md) — verified current file-transfer procedures.
- [Current State](../../CURRENT_STATE.md) — mutable project/device operational state.

## Provenance

Device identity comes primarily from the captured `ttgo.bif` and kernel device-detection line; SoC, RAM, clock, MMC, and controller observations come from Tomi's preserved kernel boot log. The Austin mapping is cross-checked against the OpenTom `GOTYPE_AUSTIN` definition and profile source. The panel string and regulatory/product identifiers are summarized in the TT3 archival runbook, with the raw diagnostic/label evidence gaps recorded in the reconciliation report. No other TomTom unit is used as evidence for TT3 hardware.
