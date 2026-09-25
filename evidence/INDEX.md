# Evidence Index

This directory indexes evidence; it is not intended to become the bulk evidence store.

## Storage rule

Large or immutable artifacts normally remain in Dropbox, NAS storage, Codex investigation directories, device images, or other established evidence locations.

Git records only what materially improves retrieval and reasoning, such as:

- artifact name / evidence-set identity
- external canonical path
- SHA-256 or other verified identity when available
- date / scope
- short statement of what the artifact proves or contains
- relationship to a canonical Git document

## Provenance rule

A current Git conclusion should point back to the strongest underlying evidence when that provenance matters. Git documentation may supersede an older operational instruction without erasing the validity of the historical evidence that produced it.

## Do not

- copy huge captures or firmware into Git merely for convenience;
- duplicate the same evidence package across multiple storage systems without a reason;
- call normalized/reformatted material byte-verbatim evidence;
- treat absence from this index as proof that evidence does not exist during bootstrap.

## TT3 identity and firmware

| Evidence set | Type / location / identity | What it contains or establishes | Canonical use / limits |
|---|---|---|---|
| TT3 archival filesystem capture, 2026-08-04 | **Primary file capture** — `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/`; critical-files archive `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/critical-files/TT3-critical-files.tar`, SHA-256 `bab2e8e7ecdb9bc478300d7760208e83f776095c210b7c0900ac900d16a0fa37`. | Capture transcript, filesystem copy, and a critical-file bundle. The bundle's three-file manifest matched all three files. Historical `ttsystem`: 4,212,715 bytes, SHA-256 `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf`. Supplementary integrity report: `/mnt/d/Codex/TT3/TT3-critical-files-inspection-report-20260806T000154Z.json`, SHA-256 `455667458c9f2016cf776a560d503d216d3c6a5533802e8309b7a624330e8a6f`. | Supports [TT3 Tomi profile](../docs/devices/tt3-tomi.md). `COMPLETE.json` says no raw whole-device image was made. `ttgo.bif` contains private device identifiers; handle this external set accordingly. The inspection report is validation, not a second primary capture. |
| Tomi live-baseline `ttsystem`, acquired 2026-08-31 | **Primary specimen** — `/mnt/d/Codex/TT3/current-live-baseline/ttsystem`; 2,219,582 bytes, SHA-256 `14505f7b17f1037163ea448ec418924d1302f575db3a7c65ef416a42af4efc21`; adjacent `ACQUISITION.txt`. | Preserves the separately acquired live-baseline binary and its transfer/acquisition identity. | Supports the specimen distinction in [TT3 Tomi profile](../docs/devices/tt3-tomi.md). It does not cryptographically bind the separately supplied 2026-09-24 kernel line to this August specimen. |
| Austin/type-42 OpenTom platform source | **Source-code working copy, not frozen** — `/mnt/d/Codex/OpenTom/src/linux-s3c24xx/linux-s3c24xx/`; `include/barcelona/gotype.h` SHA-256 `eec6f8f2451400fadf45c88ed535207bb6eeb50111d0e2757e3b5638dd610954`; `arch/arm/mach-s3c2410/tomtomgo-ioaustin.h` SHA-256 `5164d2718e0bfdc1cb53b9803ba0b6b2137ccac85bb78b1229cccdaa8bf6417d`; `arch/arm/mach-s3c2410/tomtomgo-type.c` SHA-256 `465332bd4c4d63af19bdd9828f3dbc12128f69ba12a2faa52c9aef0e43c97a0f`. | Software profile/type mapping relevant to the Austin/type-42 identity. File hashes identify the inspected bytes; they match the WSL working-copy files. | Supports the software-profile discussion in [TT3 Tomi profile](../docs/devices/tt3-tomi.md), not physical PCB identity. These files are not tracked blobs at the checkout's recorded `HEAD`; broader tree is modified, so this is not a frozen release/source snapshot. |

## Bootloader and pre-Linux

| Evidence set | Type / location / identity | What it contains or establishes | Canonical use / limits |
|---|---|---|---|
| TT3 captured `SYSTEM` / Austin 5.5279 analysis | **Primary captured package plus byte-identical static-analysis specimen** — TT3 `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/filesystem-copy/system`; TT1 comparison `/mnt/d/Codex/TT1/system-5.5279.bin`; both 1,030,991 bytes, SHA-256 `6d899cc700e25fc73e74487644458fe14984e3c26a508fc47d7f0d07f62c10da`. Analysis includes `/mnt/d/Codex/TT1/tt5279-ingress-report.md`, `/mnt/d/Codex/TT1/tt5279-disasm/`, and `/mnt/d/Codex/TT1/tt5279-usb-boot-predicate-addendum/`; decoded payload 258,408 bytes, SHA-256 `9e5072cc7ac2fa09a29182c2e66afae11c67d6625c93222cf8ed7907aa31274b`. | Establishes byte identity between Tomi's captured update package and the analyzed Austin specimen, supporting TTBL/package maps, signed-loader behavior, FAT/fixed-name selection, and bounded static boot-source findings. | Supports [Tomi Boot Chain](../docs/architecture/tomi-boot-chain.md) and [Tomi Bootloader Image Reference](../docs/reference/tomi-bootloader-image.md). This is update-package evidence, not a live NOR dump or full runtime trace. |
| TT3 historical `ttsystem` no-change reconstruction | **Reconstruction / validation set** — `/mnt/d/Codex/TT3/roundtrip-tests/tt3-nochange-20260806T004411Z/`; source specimen 4,212,715 bytes, SHA-256 `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf`. | Preserves extraction, TTBL structure comparison, payload identities, initramfs tree, and a byte-identical no-change rebuild of the historical Linux system container. | Supports the Linux-bundle map and handoff boundary in the boot-chain/image references. It does not describe the distinct 2026-08-31 live-baseline `ttsystem`. |
| Atlas III A1.0012 comparative bootloader analysis | **Cross-family comparative static analysis** — primary report `/mnt/d/Codex/TJ1/analysis/atlas3-system-bootpath-20260911/REPORT.md`; source-storage image preserved privately under `/mnt/d/Codex/TJ1/`, 1,027,604,480 bytes, SHA-256 `557a13ceab148b4e0f23922fccaa102f18b1ba362a603e8434356a6aea586e14`; extracted root-level `system` 706,801 bytes, SHA-256 `2674420245fc0f4f5d89d6c5f8542e0dcd107b136010fd010babbf3f3971bdef`. Related shared-resource comparison: `/mnt/d/Codex/TT3/tomtom-boot-drum-investigation-20260913/REPORT.md`. | Provides independent family-level evidence for TTBL update packaging, FAT/fixed-name selection, preboot USB support, Linux handoff construction, and a byte-identical startup-drum resource across distinct loader families. | Qualified comparative context only. Atlas SoC, addresses, updater behavior, USB admission semantics, and installed-NOR identity are not TT3 facts. Per-device source-image filename is intentionally omitted from this public index. |

## Linux USB role and kernel authority

| Evidence set | Type / location / identity | What it contains or establishes | Canonical use / limits |
|---|---|---|---|
| Historical/current kernel specimen set | **Preserved image specimens plus live observation** — historical TT3 `ttsystem` 4,212,715 bytes, SHA-256 `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf`; live-baseline `/mnt/d/Codex/TT3/current-live-baseline/ttsystem` 2,219,582 bytes, SHA-256 `14505f7b17f1037163ea448ec418924d1302f575db3a7c65ef416a42af4efc21`; later live kernel version string preserved separately. | Establishes distinct preserved kernel/image specimens and a separate later live-kernel observation. | Supports [Tomi Linux USB Role Kernel Reference](../docs/reference/tomi-usb-role-kernel.md). The live version string is not cryptographically bound to either preserved `ttsystem`. |
| USB-role candidate lineage V002–V007 | **Build/diagnostic candidate set** — work units under `/mnt/d/Codex/usbmode-authoritative-state-*`, `/mnt/d/Codex/usbmode-v003-redx-continuation-20260902/`, and `/mnt/d/Codex/usb-state-v007-production-20260909/`; V007 image 2,228,744 bytes, SHA-256 `344a8b733b8cdef3c58705a2a7e8039e01024db547a71773a7d8cec20cdd2825`; V007 zImage SHA-256 `b582d403d76da2da26bcc3d50b3161717e39ee4fc517c1a0bc2a376bbcab35f1`. | Preserves the serialized-authority design lineage, diagnostic evolution, notifier-subscriber defect discovery/repair, and the cleaned V007 production candidate. | Candidate/build evidence only. V007 passed offline validation but is not established as deployed or run on Tomi; earlier diagnostic findings must retain their run-specific boundaries. |
| Legacy usbmode source audit and forced-host incident | **Source correlation plus bench incident evidence** — audited Austin/OpenTom working files include `usbmode.c` SHA-256 `51a0831063085191372fca5253dd108c59207aae32780ba6a20f51690902f48d`, `s3c24xx_usbmode.c` SHA-256 `e3e6b2aaee0b3478060c7fd6c885f610d2660d6944f8ef5a718c8885d82f9614`, and `usbmode.h` SHA-256 `869e2503d528014e7ada9c078709bff2ec60996fa89432a1340d9224f30f6866`; the Tomi bench incident records an Oops/`rc=139` after a forced host-role sysfs write. | Source establishes the seven-state legacy FSM, unsafe manual sysfs shortcut semantics, and a concrete uninitialized detector-cleanup mechanism compatible with the observed crash. | Supports the permanent prohibition in [Tomi Linux USB Role Kernel Reference](../docs/reference/tomi-usb-role-kernel.md). The source tree is not proven byte-identical to the incident kernel, so the exact faulting instruction/root cause remains unproven. |

## TomiDock and USB lifecycle

| Evidence set | Type / location / identity | What it contains or establishes | Canonical use / limits |
|---|---|---|---|
| GPIO4 jumper removal and design closeout, 2026-09-24 | **Closeout / adjudication report** — `/mnt/d/Codex/TT3/20_GPIO4_JUMPER_REMOVAL_AND_DESIGN_CLOSEOUT_2026-09-24.md`; SHA-256 `22ebb2f564a19c524171886b37b970e2a7cc9c5f9a05f3097e8766777d17437b`. | Records final physical GPIO4 disposition and post-removal functional smoke results. | Supports [TomiDock Architecture](../docs/architecture/tomidock.md). Functional success is not proof of deterministic physical-detach fail-close or formal USB compliance. |
| GPIO4/BVALID lifecycle investigation, 2026-09-21 onward | **Instrumented evidence and adjudication set** — `/mnt/d/Codex/TT3/tomidock-gpio4-bvalid-audit-20260921/`; see `REPORT_PHASE4F_REDUCED_OBSERVER_DESIGN.md`, `REPORT_PHASE4G_ALTERNATING_REDUCED_OBSERVER_ADJUDICATION.md`, `phase4f_evidence/`, and `phase4g_evidence/`. The Phase 4F root, evidence, and firmware manifests and the Phase 4G evidence manifest were checked against their files. | Preserves reduced-observer design/build identity, capture packages and the alternating adjudication of lifecycle observations. | Supports [TomiDock Architecture](../docs/architecture/tomidock.md). Phase 4F is diagnostic evidence, not production firmware. The set does not establish GPIO4 as required; deterministic no-GPIO physical-detach fail-close remains unproven. |

## Power and support tooling

| Evidence set | Type / location / identity | What it contains or establishes | Canonical use / limits |
|---|---|---|---|
| PowerFlight v001.5.3 formal run, 2026-09-21 | **Formal validation package** — `/mnt/d/Codex/TT3/Evidence/powerflight-20260921-170744-7150-00.tar.gz`; SHA-256 `67a8c41caa80c09ebe2fe9114bd01ba00b5ab84343b6ef750ccb58c5e52daf8a`. All six payload files matched the embedded `manifest.sha256`. | Run logs and summary record `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, and `BATTERY_RETURN_CHILD_ATTACHED`. | Supports [TomiDock Architecture](../docs/architecture/tomidock.md) and [Current State](../CURRENT_STATE.md). One validated run; not universal electrical/charging proof and not an identification of the VBUS switch. |
| nxbattery V002 preserved build, 2026-09-16 | **Build/source/test package** — `/mnt/d/Codex/TT3/tt3-nxbattery-v002-20260916/`; package `MANIFEST.sha256` matched its listed files. Binary `build/nxbattery-v002`: 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. | Preserves the built executable, source, build record, and host tests. | Supports the nxbattery status in [Current State](../CURRENT_STATE.md). Current deployment is operator-reported; this build hash does not identify installed on-device bytes. |
| Exact-image Tomi runtime compatibility audit, 2026-09-03 | **Historical source/image-scoped validation** — `/mnt/d/Codex/usbmode-authoritative-state-v004-blackbox-compat-20260903/`; `COMMAND_COMPATIBILITY.md` SHA-256 `7e9385f4d45072f1c38df36b6429f0cfb5a34c315b1f7313e0d14511306bb183`. The report identifies the extracted `/bin/busybox` SHA-256 as `1b0d2e2d015e2b5dea4e6970645564606c88784227a2b4d90c72fc834cf52bc8`. | Records exact-image applet/shell checks run under ARM user-mode emulation with host command fallback disabled. | Historical compatibility context for [Tomi Runtime ABI](../docs/reference/tomi-runtime-abi.md), not proof of the current live BusyBox inventory or target kernel behavior. |

## Build environment

| Evidence set | Type / location / identity | What it contains or establishes | Canonical use / limits |
|---|---|---|---|
| OpenTom V002 legacy build validation, 2026-09-02 | **Build validation report** — `/mnt/d/Codex/usbmode-authoritative-state-v002-build-validation-20260902/REPORT.md`; SHA-256 `a3e82c75ef7c8c6bcdacf3a1d9c29a8b0241eb5cbf0b13645b50d8406d9776a2`. | Records a verified MacBook Debian/Bookworm chroot build context and GCC 3.3.4, with supporting logs/artifacts in its work unit. | Historical validation context for [Lab Environment](../docs/environment/lab-environment.md); does not assert that every listed path or checkout remains unchanged today. |

## Evidence gaps noted during bootstrap

- The 2026-09-24 live BusyBox inventories are preserved in a Codex attachment, but no durable raw capture was found. The inventories in [Tomi Runtime ABI](../docs/reference/tomi-runtime-abi.md) remain operator-provided and are not hash-bound to the live binaries.
- The successful 2026-09-24 SCP proof is described in the operator brief, but its raw transcript, transferred-file byte count, and digest were not found as a durable artifact. The [Tomi File Transfer runbook](../docs/runbooks/tomi-file-transfer.md) is procedure, not proof of that transfer.
- The current OpenTom Austin/type-42 source files are present in the working checkout, but were not found as tracked blobs at its recorded `HEAD`; no frozen source snapshot was identified during this pass. They are not indexed as immutable source evidence.
- Exact installed-byte identities for current nxbattery, `tomidock-netd`, `/etc/rc`, and the ESP image remain unbound in the available deployment records; see [Current State](../CURRENT_STATE.md).

This is a bootstrap selection of high-value evidence for material already migrated into LUCE, not a complete inventory of `/mnt/d/Codex`. Later corpus-coverage work may add evidence for other domains.
