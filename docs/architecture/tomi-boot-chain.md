# Tomi Boot Chain

## Scope

This page records the best-supported TT3 “Tomi” path from pre-Linux startup to the Linux image boundary. It separates device/profile evidence from static analysis of a matching bootloader update package. It does not claim that the update package is a byte-for-byte dump of the bootloader currently executing from nonvolatile boot storage.

## Established stages and evidence boundary

The earliest Tomi-specific pre-Linux stage established by the preserved material is a Samsung/Austin-family TomTom bootloader identified as `s3c24xx` version `5.5279`. Tomi's captured `bootloaderversion.txt` reports `5.5279`; `ttgo.bif` independently reports numeric `55279`. The silicon mask-ROM/reset stage before this bootloader has not been independently characterized for Tomi and is intentionally left unnamed here.

The root-level `SYSTEM` file in Tomi's 2026-08-04 filesystem capture is 1,030,991 bytes and is byte-identical to the analyzed Austin `system-5.5279.bin` specimen. That establishes that the captured filesystem contains the same update-package bytes analyzed in the Austin static audit. It does not establish that those bytes are identical to the active bootloader image in NOR or otherwise prove the current bootloader's complete runtime behavior.

## Bootloader storage and selection path

Static re-audit of that exact 5.5279 package corrected the earlier board-profile attribution: **Austin/type 42 is constructed at `0x30b1fad4` and selects legacy MOVINAND/iNAND SDI plus the full-speed USB device controller at `0x52000000`; `0x30b1fe24` is Bergamo/type 43 and selects the previously cited high-speed paths.** Austin's normal package source remains the internal MMC/SDI block device with FAT16/32. The selector tries fixed 8.3 names including `DIAGSYS`, `SIGNAPPSIGN`, `SYSTEM`, `TTSYSTEM`, `LTSYSTEM`, and `CMDLINE.TXT`; the `SYSTEM` version probe reads the six-byte version at file offset `0x3fffc` and influences selection among standard package names. A first-section length other than `0x40000` selects the update attempt rather than acting as a universal rejection.

For Tomi, this is strong package/platform-correlated evidence, not an independently instrumented trace of a particular power-on. A later read-only TT3 sector acquisition and replay of the Austin directory lookup against one captured V003 storage state confirmed that an entry with the Linux VFAT dummy short-name field could be skipped before the integrity-checked TTBL loader opened `TTSYSTEM`; that is a historical specimen-specific observation, not a statement about the current directory. The TT3 evidence identifies Linux-visible main storage as `/dev/mmcblk0` and the Austin software profile as S3C2412, but does not expose a normal live bootloader read trace. USB MSC in the analyzed bootloader is device/target-side service, not evidence of booting directly from a USB stick. Likewise, the bounded search found no grounded USB-host boot path; this is a static-analysis result for the specimen, not an absolute claim about every possible build.

The follow-on CMDLINE bounds audit closes the local-copy question for this package. The reader at `0x30b0d9dc` clears a 1,024-byte source buffer, accepts declared file lengths below `0x400`, and normalizes bytes below `0x20` to NUL. Bounded selector tests reached that reader before the first image-load attempt for the tested `DIAGSYS`, `SIGNAPPSIGN`, `SYSTEM`, `TTSYSTEM`, `LTSYSTEM`, all-load-fail and USB-priority cases. `LTSYSTEM` still follows its separate reset/update route rather than the ordinary Linux parameter-builder handoff.

## Linux image path and handoff

The historical TT3 `ttsystem` specimen is a separate 4,212,715-byte `TTBL` container, not the bootloader image. Its first section is loaded at `0x31700000` and contains a small wrapper/decompressor followed by a gzip-compressed Linux kernel. Its second section is a gzip initramfs loaded at `0x31000000`. The container terminator records entry value `0x31700000` and parameter value `0x30000000`. A no-change extraction/rebuild reproduced the complete file byte-for-byte, both payloads, the initramfs tree, and the TTBL structure.

The 5.5279 loader's generic TTBL routine is directly observed in the matching `SYSTEM` update-package disassembly: it validates `TTBL`, loads section payloads to file-declared addresses, checks each payload with MD5 plus an embedded-key Blowfish integrity tag, obtains an entry address and parameter from the terminator, constructs boot parameters, and transfers control at **`0x30b050f4: bx r3`**. The payload tag does **not** authenticate the destination address, final entry or parameter address; bounded original-instruction tests accepted changed addresses with unchanged valid payload/tag bytes. No reached public-key/vendor-authenticity gate or entry-range check was found in this generic handoff.

The ordinary parameter builder contains a 32-byte `ATAG_CMDLINE` payload followed immediately by the eight-byte zero-sized `ATAG_NONE` terminator. Its NUL-terminated CMDLINE copy has no destination-length argument. Thirty-one data bytes plus NUL are the strict contained maximum. At effective length `S=32`, the terminating NUL is written one byte past the CMDLINE payload into an already-zero first byte of `ATAG_NONE.size`, so no value changes in the original template. Effective terminator corruption begins at `S=33`; the copy reaches saved `r4` at `S=40`, saved `lr` at `S=44`, and the caller frame at `S=48`.

The builder exports exactly `0xdc` bytes from its local parameter template to the TTBL trailer-selected parameter address, ending at the `ATAG_NONE` header. Thus CMDLINE-derived ATAG corruption can reach the outgoing Linux parameter block, while saved-register and caller-frame corruption remain local to the builder frame. Matching OpenTom kernel source walks tags until a zero `hdr.size`, so corruption beginning at `S=33` removes the intended terminator; exact downstream behavior of malformed lists is not established. The normal tested path reaches `0x30b050f4: bx r3` before the builder restores its saved registers, and the bounded audit deliberately did not execute the loaded entry or return through a corrupted saved LR. These findings establish memory and parameter-construction bounds, **not a post-handoff control-flow consequence or exploit path**.

Do not treat the update package's own entry (which dispatches to its section 2) as the Linux handoff merely because its numeric value matches the historical Linux bundle entry. The matching TT3 `ttsystem` layout supports this separate chain: bootloader selects and validates the Linux bundle; control enters the first loaded wrapper at `0x31700000`; that wrapper inflates the kernel and transfers to fixed decompressed Linux entry `0x30008000`, while the initramfs is the second container payload.

A distinct S3C2412 retained-state path bypasses FAT/TTBL loading: when INFORM0 selects retained state and INFORM1 is not sentinel `0x8024`, the bootstrap restores state and executes **`0x0000a550: mov pc,r0`** using INFORM1 as the resume address. OpenTom suspend/resume source writes the physical resume address into INFORM1, so this is best classified as the intended resume contract, not a USB/UART downloader. Physical retained-RAM behavior on Tomi remains unqualified.

The separate live-baseline `ttsystem` acquired 2026-08-31 is 2,219,582 bytes and has a different SHA-256 from the historical specimen. It must not inherit the historical specimen's section offsets or payload map without its own analysis. The current live Linux release line is recorded in the [Tomi device profile](../devices/tt3-tomi.md); it is not cryptographically tied to either `ttsystem` file by the available evidence.

## Update and recovery relationship

The `SYSTEM` file is an update/package specimen containing a 5.5279 bootloader payload; it is not interchangeable with the Linux `ttsystem` bundle. Its selected updater ultimately resets after programming logic. Successful `LTSYSTEM` loading also requests reset rather than taking the ordinary TTBL trailer-entry handoff. USB recovery is device-side MSC that persists blocks to internal storage for later FAT/TTBL selection; no direct USB/UART download-and-go path was established. These forensic findings are not an adopted bootloader-writing or recovery procedure. This page provides no flashing instructions.

## Established versus inferred behavior

- **Direct TT3 identity evidence:** bootloader version strings, TT3 root-level `SYSTEM` bytes, historical `ttsystem` bytes, and Tomi Linux boot/storage observations.
- **Direct binary observations:** the matching `SYSTEM` package's bootstrap, corrected Austin storage/USB registration, FAT/name selector, CMDLINE reader and exact parameter-copy bounds, TTBL integrity loader/trust boundary, retained-RAM resume branch, updater/reset paths, and handoff code.
- **Reconstructed-container evidence:** historical TT3 `ttsystem` section map, payload identities, and byte-identical no-change round trip.
- **Inference with limits:** these artifacts together support the stated Tomi boot path, but no preserved trace observes every branch and address during a current physical boot, and no acquired NOR image binds the active bootloader to the update-package specimen.

## Comparative / family-level findings

An independently analyzed TomTom ONE v8 / SiRF Atlas III specimen provides useful family context, not direct evidence of Tomi execution. Its root-level `system` is a distinct ten-section TTBL update package containing an A1.0012 (`clist` 216525) bootloader, a NOR updater, and board resources. The extracted loader uses FAT and fixed boot names including `DIAGSYS`, `SIGNAPPSIGN`, `SYSTEM`, `TTSYSTEM`, `LTSYSTEM`, and `CMDLINE.TXT`; it parses TTBL packages, includes preboot USB support, and constructs Linux ATAGs. This corroborates a broader TomTom bootloader architecture, while its SoC, addresses, memory placement, updater, and unresolved USB admission predicate remain Atlas-specific. In particular, Austin 5.5279's exact USB-storage predicate and any numeric address or branch must not be transferred to Atlas or used as evidence about another loader.

The same Atlas III 1.0012 specimen independently calls an embedded startup-drum player before Linux handoff. The boot-drum comparison found its 25,157-byte compressed sample byte-identical to the analyzed Austin 5.5279 sample, despite materially different SoC audio backends. This supports a shared product-level sample/startup convention, not identity of the loaders or proof of Tomi's live NOR contents. Provenance: `/mnt/d/Codex/TJ1/analysis/atlas3-system-bootpath-20260911/REPORT.md` and `/mnt/d/Codex/TT3/tomtom-boot-drum-investigation-20260913/REPORT.md`.

In the analyzed Austin 5.5279 and Atlas III 1.0012 normal startup paths, execution calls that embedded-sample player unconditionally at the observed call site. The reviewed disassembly found no software mute/volume, power, USB, storage, reset-cause, or power-button gate around that normal-path request, and audible playback is not used as a boot-health predicate. Therefore an isolated missing startup drum while boot otherwise continues normally has no supported diagnostic significance from these two analyzed paths alone. The hardware reason for requested-but-inaudible output remains unknown, the active Tomi NOR loader is not byte-bound to the captured `SYSTEM` package, and alternate startup paths or other loader versions are not covered by this finding.

## Known unknowns

- Contents and control flow of any earlier SoC ROM/reset stage.
- Byte identity of the executing Tomi bootloader in NOR versus the captured `SYSTEM` update package.
- Which boot filename/fallback branch was taken on any particular boot.
- Complete interpretation of the remaining `SYSTEM` version-probe return codes, downstream Linux behavior for malformed CMDLINE-derived ATAG lists, and real filesystem short/error-read semantics.
- Complete partitionless legacy-MSC initialization behavior beyond the demonstrated missing-image/red-X cases.
- Physical retained-RAM resume behavior on current Tomi hardware/software.
- Final decompressor-to-kernel handoff details and exact meaning of the historical container parameter `0x30000000`.
- Internal section/padding fields in the separate 2026-08-31 live-baseline `ttsystem`.

## Canonical references

- [Tomi device profile](../devices/tt3-tomi.md) — hardware and firmware identity, including the two distinct `ttsystem` specimens.
- [Tomi bootloader image reference](../reference/tomi-bootloader-image.md) — specimen hashes, byte maps, CMDLINE/ATAG bounds, formats, and extraction boundaries.
- [Evidence Index](../../evidence/INDEX.md) — selected durable evidence anchors.

## Provenance

Primary inputs are the TT3 2026-08-04 filesystem capture and critical-files bundle, the TT3 no-change `ttsystem` round-trip package, the earlier TT1/Austin 5.5279 investigations, the 2026-09-28 independent execution-path re-audit at `/mnt/d/Codex/TT3/s55279-execution-path-reaudit-20260928/REPORT.md`, and the follow-on CMDLINE bounds set at `/mnt/d/Codex/TT3/s55279-cmdline-bounds-audit-20260928/`. The execution-path re-audit re-extracted and re-traced the byte-identical TT1/TT3 `SYSTEM` package, correcting the Austin/Bergamo constructor attribution while confirming the core FAT/TTBL and USB-latch findings. The CMDLINE audit executed bounded original instructions with synthetic data and explicit substitutions, preserving exact boundary results while stopping before loaded-entry execution or corrupted-return control flow. The Austin binary findings remain package/binary evidence and are not promoted to live-NOR observations.