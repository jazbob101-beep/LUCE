# Tomi Boot Chain

## Scope

This page records the best-supported TT3 “Tomi” path from pre-Linux startup to the Linux image boundary. It separates device/profile evidence from static analysis of a matching bootloader update package. It does not claim that the update package is a byte-for-byte dump of the bootloader currently executing from nonvolatile boot storage.

## Established stages and evidence boundary

The earliest Tomi-specific pre-Linux stage established by the preserved material is a Samsung/Austin-family TomTom bootloader identified as `s3c24xx` version `5.5279`. Tomi's captured `bootloaderversion.txt` reports `5.5279`; `ttgo.bif` independently reports numeric `55279`. The silicon mask-ROM/reset stage before this bootloader has not been independently characterized for Tomi and is intentionally left unnamed here.

The root-level `SYSTEM` file in Tomi's 2026-08-04 filesystem capture is 1,030,991 bytes and is byte-identical to the analyzed Austin `system-5.5279.bin` specimen. That establishes that the captured filesystem contains the same update-package bytes analyzed in the Austin static audit. It does not establish that those bytes are identical to the active bootloader image in NOR or otherwise prove the current bootloader's complete runtime behavior.

## Bootloader storage and selection path

Static disassembly of that exact 5.5279 package, under its Austin/type-42 configuration, establishes a direct boot-source path through the registered `HSMOVINAND` block device and a FAT16/32 filesystem. The selector tries fixed 8.3 names including `DIAGSYS`, `SIGNAPPSIGN`, `SYSTEM`, `TTSYSTEM`, `LTSYSTEM`, and `CMDLINE.TXT`; the `SYSTEM` version probe reads the six-byte version at file offset `0x3fffc` and influences selection among standard package names. The precise product-policy meaning of every probe result is not named by the binary.

For Tomi, this is strong package/platform-correlated evidence, not an independently instrumented trace of a particular power-on. A later read-only TT3 sector acquisition and replay of the Austin directory lookup against one captured V003 storage state confirmed that an entry with the Linux VFAT dummy short-name field could be skipped before the signed loader opened `TTSYSTEM`; that is a historical specimen-specific observation, not a statement about the current directory. The TT3 evidence identifies Linux-visible main storage as `/dev/mmcblk0` and the Austin software profile as S3C2412, but does not expose a normal live bootloader read trace. USB MSC in the analyzed bootloader is device/target-side service, not evidence of booting directly from a USB stick. Likewise, the bounded search found no grounded USB-host boot path; this is a static-analysis result for the specimen, not an absolute claim about every possible build.

## Linux image path and handoff

The historical TT3 `ttsystem` specimen is a separate 4,212,715-byte `TTBL` container, not the bootloader image. Its first section is loaded at `0x31700000` and contains a small wrapper/decompressor followed by a gzip-compressed Linux kernel. Its second section is a gzip initramfs loaded at `0x31000000`. The container terminator records entry value `0x31700000` and parameter value `0x30000000`. A no-change extraction/rebuild reproduced the complete file byte-for-byte, both payloads, the initramfs tree, and the TTBL structure.

The 5.5279 loader's generic TTBL routine is directly observed in the matching `SYSTEM` update-package disassembly: it validates `TTBL`, loads section payloads to declared addresses, verifies section signatures, obtains an entry address and parameter from the terminator, constructs boot parameters, and branches to the entry. Do not treat the update package's own entry (which dispatches to its section 2) as the Linux handoff merely because its numeric value matches the historical Linux bundle entry. The matching TT3 `ttsystem` layout supports this separate chain: bootloader selects and validates the Linux bundle; control enters the first loaded wrapper at `0x31700000`; that wrapper contains the compressed kernel, while the initramfs is the second container payload. The final wrapper-to-decompressed-kernel register/argument details are not fully established in the preserved TT3 evidence.

The separate live-baseline `ttsystem` acquired 2026-08-31 is 2,219,582 bytes and has a different SHA-256 from the historical specimen. It must not inherit the historical specimen's section offsets or payload map without its own analysis. The current live Linux release line is recorded in the [Tomi device profile](../devices/tt3-tomi.md); it is not cryptographically tied to either `ttsystem` file by the available evidence.

## Update and recovery relationship

The `SYSTEM` file is an update/package specimen containing a 5.5279 bootloader payload; it is not interchangeable with the Linux `ttsystem` bundle. The static loader's signed-package mechanism and file selection are forensic findings, not an adopted or validated bootloader-writing procedure. This page provides no flashing or recovery instructions.

## Established versus inferred behavior

- **Direct TT3 identity evidence:** bootloader version strings, TT3 root-level `SYSTEM` bytes, historical `ttsystem` bytes, and Tomi Linux boot/storage observations.
- **Direct binary observations:** the matching `SYSTEM` package's bootstrap, Austin storage registration, FAT/name selector, signed-TTBL loader, and handoff code.
- **Reconstructed-container evidence:** historical TT3 `ttsystem` section map, payload identities, and byte-identical no-change round trip.
- **Inference with limits:** these artifacts together support the stated Tomi boot path, but no preserved trace observes every branch and address during a current physical boot, and no acquired NOR image binds the active bootloader to the update-package specimen.

## Comparative / family-level findings

An independently analyzed TomTom ONE v8 / SiRF Atlas III specimen provides useful family context, not direct evidence of Tomi execution. Its root-level `system` is a distinct ten-section TTBL update package containing an A1.0012 (`clist` 216525) bootloader, a NOR updater, and board resources. The extracted loader uses FAT and fixed boot names including `DIAGSYS`, `SIGNAPPSIGN`, `SYSTEM`, `TTSYSTEM`, `LTSYSTEM`, and `CMDLINE.TXT`; it parses TTBL packages, includes preboot USB support, and constructs Linux ATAGs. This corroborates a broader TomTom bootloader architecture, while its SoC, addresses, memory placement, updater, and unresolved USB admission predicate remain Atlas-specific. In particular, Austin 5.5279's exact USB-storage predicate and any numeric address or branch must not be transferred to Atlas or used as evidence about another loader.

The same Atlas III 1.0012 specimen independently calls an embedded startup-drum player before Linux handoff. The boot-drum comparison found its 25,157-byte compressed sample byte-identical to the analyzed Austin 5.5279 sample, despite materially different SoC audio backends. This supports a shared product-level sample/startup convention, not identity of the loaders or proof of Tomi's live NOR contents. Provenance: `/mnt/d/Codex/TJ1/analysis/atlas3-system-bootpath-20260911/REPORT.md` and `/mnt/d/Codex/TT3/tomtom-boot-drum-investigation-20260913/REPORT.md`.

## Known unknowns

- Contents and control flow of any earlier SoC ROM/reset stage.
- Byte identity of the executing Tomi bootloader in NOR versus the captured `SYSTEM` update package.
- Which boot filename/fallback branch was taken on any particular boot.
- Complete interpretation of the `SYSTEM` version-probe return codes and `CMDLINE.TXT` consumption semantics.
- Final decompressor-to-kernel handoff details and exact meaning of the historical container parameter `0x30000000`.
- Internal section/padding fields in the separate 2026-08-31 live-baseline `ttsystem`.

## Canonical references

- [Tomi device profile](../devices/tt3-tomi.md) — hardware and firmware identity, including the two distinct `ttsystem` specimens.
- [Tomi bootloader image reference](../reference/tomi-bootloader-image.md) — specimen hashes, byte maps, formats, and extraction boundaries.
- [Evidence Index](../../evidence/INDEX.md) — selected durable evidence anchors.

## Provenance

Primary inputs are the TT3 2026-08-04 filesystem capture and critical-files bundle, the TT3 no-change `ttsystem` round-trip package, and the TT1/Austin 5.5279 ingress report plus bounded disassembly. The TT3 `SYSTEM` and TT1 analysis input were independently hashed and are byte-identical. Exact paths and scope limitations are maintained in the external migration reconciliation report; the Austin binary findings are not promoted to live-NOR observations.
