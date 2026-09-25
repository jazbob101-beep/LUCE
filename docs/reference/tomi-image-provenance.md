# Tomi Firmware and Image Provenance

## Scope

This reference describes the identity, origin, transformation, and evidentiary limits of material TT3 “Tomi” firmware and storage specimens. It is a provenance ledger, not a bootloader analysis, recovery procedure, evidence-index replacement, or deployment guide. The [Evidence Index](../../evidence/INDEX.md) remains the retrieval guide; [Tomi Disaster Recovery](../runbooks/tomi-disaster-recovery.md), [Tomi File Transfer](../runbooks/tomi-file-transfer.md), [Tomi Boot Chain](../architecture/tomi-boot-chain.md), and [Tomi Bootloader Image](tomi-bootloader-image.md) own their respective operational and technical details.

The evidence was reconciled from the indexed TT3 captures, raw-image acquisitions, TTBL round-trip, bootloader work, current/live-baseline transfer, USB-role V002–V007 lineage, and V003 recovery/namespace investigations. The associated cross-device bootloader comparison is included only where it changes the interpretation of a Tomi specimen.

## Terminology and storage boundaries

| Term | Use in this reference |
|---|---|
| **Acquisition / capture** | A recorded collection event. A file-level capture, block-device image, and individual file acquisition are different acquisition products. |
| **Specimen** | A byte-identified artifact with stated origin and limits. A hash identifies bytes, not their physical origin or execution. |
| **Image** | Reserved for a raw storage address-space image, a partition image, or a specifically named firmware candidate. Do not use it as a synonym for a directory copy or `ttsystem`. |
| **Container / package / payload** | `ttsystem` is a Linux TTBL container; root-level `SYSTEM` is a bootloader update package; a decoded member is a derived payload. They are not interchangeable. |
| **Baseline** | A particular known specimen used as a comparison or restore source, not an assertion that it is currently installed. |
| **Candidate / reconstruction** | A constructed artifact or rebuilt representation. A successful offline validation or byte-identical no-change reconstruction does not establish deployment. |
| **Live / current / deployed** | *Live observation* is an observation at a time; *current installed bytes* require byte identity bound to the device; *deployed* requires supported installation evidence. These terms are not inferred from a version label or hash alone. |
| **Donor / comparison** | Another device's material may corroborate shared packaging or byte identity, but does not establish Tomi's installed state or hardware-specific behavior. |

Storage domains must remain separate: removable/internal MMC; VFAT filesystem contents; internal NOR/MTD including the observed NGFFS area; and volatile runtime RAM. The August 2026 raw image covers the MMC device address space only. No single acquisition here images all nonvolatile Tomi storage.

## Primary TT3 acquisitions

### 2026-08-04 file-level capture

`/mnt/d/Codex/TT3/TT3-capture-20260804-163938/` is a verified filesystem/file capture (647 files, approximately 3.5 GiB), not a raw block image. Its `COMPLETE.json` explicitly records `RawImageCreated: false`. It preserves root-level files including `system` and `ttsystem`, filesystem metadata, manifests, and capture records. Do not call it a factory image or whole-device image.

### 2026-08-05 raw MMC acquisition

`/mnt/d/Codex/TT3/TT3-raw-image-20260805-084600/TT3-20260805-084600.img` is 4,001,366,016 bytes with SHA-256 `066be567900fa81c403d79146dd4f68d74fad3390f0fc7c213205b29768a4100`. The acquisition assessment records completion at 2026-08-05 08:46 -05, ddrescue with zero read errors/bad sectors/bad areas, and a local image hash. Its 612-file comparison with the prior capture found 608 exact matches and four mutable-file differences. A later source-device hash followed a normal boot and therefore is not a contemporaneous bit-for-bit source comparator. Classification: complete and locally integrity-verified raw MMC acquisition; exact source-state byte equivalence is not independently established. It does not include a NOR/NGFFS dump or RAM.

The earlier file capture and later raw image are complementary, distinct specimens. The file capture’s “no raw image” statement applies to that August 4 acquisition, not to the subsequent August 5 acquisition.

### 2026-08-31 live-baseline file acquisition

`/mnt/d/Codex/TT3/current-live-baseline/ttsystem` is 2,219,582 bytes, SHA-256 `14505f7b17f1037163ea448ec418924d1302f575db3a7c65ef416a42af4efc21`, MD5 `1279211cc7c9bbefe5e05a1aeb2916f4`. `ACQUISITION.txt` records transfer from the running Tomi via active FTP through the ESP mapping. This is a separately acquired file-level baseline, not a block image and not the August 4 historical container. Its acquisition does not establish that these bytes remain installed today.

## Specimen ledger

Hashes and sizes below identify preserved bytes as documented in the corresponding acquisition, round-trip, candidate, or canonical reports. Acquisition context and deployment evidence are deliberately separate fields. “Not established” means the reviewed record does not bind the artifact to a current device state.

| Specimen | Class / path | Identity | Origin, transformation, and contents | Use and limits / deployment |
|---|---|---|---|---|
| TT3 file-level capture | `FILESYSTEM_CAPTURE`; `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/` | 647 files; ~3.5 GiB corpus | Tomi filesystem/file capture with metadata and capture records; `COMPLETE.json`: `RawImageCreated=false`. | Historical file state; not whole-device, factory image, or live current state. |
| TT3 raw MMC | `RAW_STORAGE_IMAGE`; `/mnt/d/Codex/TT3/TT3-raw-image-20260805-084600/TT3-20260805-084600.img` | 4,001,366,016 bytes; SHA-256 `066be567900fa81c403d79146dd4f68d74fad3390f0fc7c213205b29768a4100` | Complete ddrescue capture; zero reported errors; locally hashed; differs from prior file capture in four mutable files. | Whole MMC address-space evidence. Exact source-state byte equivalence not independently verified; excludes NOR/NGFFS and RAM; not current device identity. |
| Historical TT3 `ttsystem` | `EXTRACTED_FILE` / Linux `TTBL_CONTAINER`; `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/filesystem-copy/ttsystem` | 4,212,715 bytes; SHA-256 `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf` | Extracted as a file from the August 4 capture; contains the historical Linux kernel/initramfs container. | Valid for historical bundle/layout analysis and round-trip; distinct from live baseline and all candidates; not established currently installed. |
| TT3 root-level `system` | `EXTRACTED_FILE` / `BOOTLOADER_UPDATE_PACKAGE`; `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/filesystem-copy/system` | 1,030,991 bytes; SHA-256 `6d899cc700e25fc73e74487644458fe14984e3c26a508fc47d7f0d07f62c10da` | Captured filesystem file; 5.5279 TTBL update package. Byte-identical to the analyzed TT1/Austin package. | Package/static-analysis evidence only; not live NOR, Linux `ttsystem`, or whole-device image. Active NOR identity is unknown. |
| Decoded 5.5279 payload | `DECODED_PAYLOAD`; `/mnt/d/Codex/TT1/tt5279-usb-boot-predicate-addendum/work/bootloader.decoded` | 258,408 bytes; SHA-256 `9e5072cc7ac2fa09a29182c2e66afae11c67d6625c93222cf8ed7907aa31274b` | Gzip-inflated member derived from captured `system`, runtime base `0x30b00000`. | Allows bounded static analysis of that package member; not a separately acquired bootloader or proof of current NOR execution. |
| No-change reconstructed `ttsystem` | `RECONSTRUCTED_CONTAINER`; `/mnt/d/Codex/TT3/roundtrip-tests/tt3-nochange-20260806T004411Z/rebuilt/ttsystem` | 4,212,715 bytes; SHA-256 `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf` | Extracted historical bundle, rebuilt with preserved toolchain, then compared; whole output, kernel/initramfs payloads, initramfs tree and TTBL structure were byte/identity checked. | Byte-identical reconstruction validates this tooling/format path. It is not a new primary acquisition or deployment. |
| Live-baseline `ttsystem` | `LIVE_ACQUISITION`; `/mnt/d/Codex/TT3/current-live-baseline/ttsystem` | 2,219,582 bytes; SHA-256 `14505f7b17f1037163ea448ec418924d1302f575db3a7c65ef416a42af4efc21`; MD5 `1279211cc7c9bbefe5e05a1aeb2916f4` | Separately transferred from Tomi on 2026-08-31; different specimen and size from historical container. | Exact preserved file identity and V002 restore baseline per operator report; current installed identity is not re-hashed. |
| V002 role-authority candidate | `PATCHED_CANDIDATE_IMAGE`; `/mnt/d/Codex/usbmode-authoritative-state-v002-image-validation-20260902/review/ttsystem.usbmode-authority-v002` | 2,228,840 bytes; SHA-256 `8755e8243778bcbc2e8c4820c2677f6dbc5eccf84c5d4e90995f4b7fd551a0c4`; compiled `vmlinux` 3,038,203 bytes, SHA-256 `da810529e7678069dd0169dc6e9e87b62a3ac6d247ac1b68b54741836ea34a46` | Reconstructed from live-baseline lineage with authoritative USB-role implementation and startup correction; offline image validation passed. | `PATCHED_CANDIDATE`, `OFFLINE_VALIDATED`, `BENCH_RUN` (operator-reported), followed by operator-reported recovery to baseline. Bench history does not hash-bind current installed bytes. Candidate is not a recovery specimen. |
| V003 diagnostic candidate | `PATCHED_CANDIDATE_IMAGE`; `/mnt/d/Codex/usbmode-authoritative-state-v003-blackbox-20260902/review/ttsystem.usbmode-authority-v003-diag` | 2,231,621 bytes; SHA-256 `cc7173c72bcbe5974940ac95685c9ff128807ea9ceed41045e7c10c63dc28d02`; zImage 1,094,332 bytes; SHA-256 `10e2ddf79f341bbd247151a5cc2b3b5f3d6bf55cbbe60558595e2fafd6c94c6d` | V002-lineage instrumentation-only derivative; kernel/modules frozen; rootfs logging added. | `DIAGNOSTIC_CANDIDATE`, `BENCH_RUN` reported, `DEPLOYMENT_UNKNOWN`: reports include red-X and later differing boot observations. Subsequent raw namespace acquisition found baseline under active `ttsystem` and V003 bytes under a longer alternate name; chronology and which bytes booted are unresolved. |
| V004 logger compatibility candidate | `PATCHED_CANDIDATE_IMAGE`; `/mnt/d/Codex/usbmode-authoritative-state-v004-blackbox-compat-20260903/review/ttsystem.usbmode-authority-v004-diag` | 2,231,793 bytes; SHA-256 `45ad091ce4a5ad4c500f37a9bf601b9719019e706bd31eec3cc825463453aeb7` | V003-derived shell/logger compatibility correction; USB policy unchanged. | `DIAGNOSTIC_CANDIDATE`, offline validation; `NOT_DEPLOYED` in reviewed record. |
| V005 input-to-worker trace candidate | `PATCHED_CANDIDATE_IMAGE`; `/mnt/d/Codex/usb-state-v005-paint-the-teeth-20260903/review/ttsystem.usbmode-authority-v005-trace` | 2,233,931 bytes; SHA-256 `1398b7cf9016ccd32ac7674b2735ac31c0a168a6843da19e09ef61ecef124e52` | V004-lineage diagnostic trace through raw input, notifier, queue/worker and admission. | `DIAGNOSTIC_CANDIDATE`, offline validation; `NOT_DEPLOYED` in reviewed record. RUN-3 evidence later exposed notifier-list reset. |
| V006 notifier repair candidate | `PATCHED_CANDIDATE_IMAGE`; `/mnt/d/Codex/usb-state-v006-notifier-repair-20260903/review/ttsystem.usbmode-authority-v006-notifier-fix` | 2,234,015 bytes; SHA-256 `066debd33caf8425690d5ef25e5ac20fee9e38733e2f60c30b58f77fb946f4d1` | Built one-line buspower-notifier lifecycle repair retaining V005 diagnostics. | `PATCHED_CANDIDATE`, `OFFLINE_VALIDATED`, `NOT_DEPLOYED`; hardware result untested. |
| V007 production-lineage candidate | `PATCHED_CANDIDATE_IMAGE` / `BUILD_ARTIFACT`; `/mnt/d/Codex/usb-state-v007-production-20260909/review/ttsystem.usbmode-authority-v007-production` | `ttsystem` 2,228,744 bytes; SHA-256 `344a8b733b8cdef3c58705a2a7e8039e01024db547a71773a7d8cec20cdd2825`; zImage SHA-256 `b582d403d76da2da26bcc3d50b3161717e39ee4fc517c1a0bc2a376bbcab35f1`; vmlinux SHA-256 `2827cee96f367f3518b2ac28350d0a1f8fa6d6d1d9e7de8b6382b1e54daff882` | Production-lineage candidate based on V002 role authority and V006 notifier repair, with diagnostic tracing removed. | `PATCHED_CANDIDATE`, `OFFLINE_VALIDATED`, `NOT_DEPLOYED`; no V007 Tomi run is established. Highest version does not mean current. |
| V003 recovery-sector evidence | `PARTIAL RAW STORAGE ACQUISITION`; `/mnt/d/Codex/usbmode-v003-redx-continuation-20260902/` | 8,800,768 raw bytes in the recorded acquisition; not a whole-device image | Read-only host-side storage/namespace investigation. Baseline bytes were found as active `ttsystem`; V003 bytes were separately located under a longer name; a valid uppercase SFN was constructed during the repair investigation. | Evidence of the acquired interval/name state only. Filesystem clean-fsck gate failed and physical reboot was not performed; chronology/recovery success remain unproven. |
| Rescue `.good` / `.failed` / staged files | `PHYSICAL_DEVICE_STATE` or transient stage, not independently acquired specimens | No separately verified current byte/hash identity for `.tomi-rescue/ttsystem.good` or stage files. `.failed` was reported as matching V002 but was not independently acquired from the device in reviewed evidence. | Rescue work describes a V002 failure copy and restoration of the exact 2026-08-31 baseline; recovery procedure is owned by the disaster-recovery runbook. | `.good` identity, sidecar, live directory state, and stage identities remain unbound. Operator-reported successful V002 recovery is not an independent file acquisition. |

## Transformation chains

```text
2026-08-04 filesystem capture
  ├── extracted file: historical ttsystem (4,212,715 bytes)
  │     ├── extract TTBL/kernel/initramfs representations
  │     └── no-change rebuild → byte-identical reconstructed ttsystem
  └── extracted file: root-level system (5.5279 update package)
        └── gzip decode → 258,408-byte derived bootloader payload
```

The historical `ttsystem` round-trip validates the particular extraction/rebuild chain, not the separate 2026-08-31 live baseline. V002–V007 are modified/reconstructed candidate lineage, not transformations of the historical file capture. The USB-role reference records their specific source and component relationships.

## Recovery-related artifacts

The V002 bench report records an operator-observed rescue: failed V002 retained as `.tomi-rescue/ttsystem.failed`, baseline restored, and maintenance marker cleared. The acquired August 31 baseline is hash-bound as an external file, but the on-device `.good` copy, its sidecar, `.failed`, and intermediate staging files were not separately acquired and verified as current independent specimens. Treat their names and claimed hashes as report/runtime evidence, not a new primary acquisition.

The later V003 filename/SFN work acquired a limited raw storage interval and distinguished the active baseline file from V003 bytes under a longer alternate filename. A fresh uppercase `TTSYSTEM` SFN was produced in the host-side namespace work, but `fsck.fat -n` returned 1 for unrelated inconsistencies and no physical reboot followed. This does not establish V003 as booted, installed, or recovered. Operational mechanics remain in the recovery runbook.

## Current versus historical identity

| Identity category | What is established |
|---|---|
| Current/live observation | A 2026-09-24 Linux version string and other runtime observations are recorded separately in canon; they are not cryptographic hashes of a kernel specimen. |
| Current installed `ttsystem` | Not currently hash-bound by the reviewed evidence. The August 31 live-baseline file is the last independently acquired `ttsystem` specimen, not proof of today’s active file. |
| Operator-reported deployment | Current nxbattery V002 is reported deployed/running, but installed bytes are not matched to the saved binary hash. TomiDock `tomidock-netd`, `/etc/rc`, and ESP image also lack exact current hash binding. These are software deployments, not evidence about Tomi’s `ttsystem`. |
| Candidate artifacts | V002–V007 have preserved candidate identities. V007 is explicitly not established as deployed. |
| Internal NOR / NGFFS | No raw live NOR image is available; the update-package `SYSTEM` file is not a NOR dump. `mtd0`/NGFFS observations do not provide a byte image. |
| Historical capture | The August 4 filesystem capture and August 5 MMC image preserve past states; neither should be labeled current. |

## Source and donor boundaries

The WSL OpenTom working tree `/home/jazbob/opentom` is at Git HEAD `eb3d2037315bde5da2875b38cba4efba9d47eac7`, with modified and untracked files. It is a dirty working tree, not a frozen release snapshot. No source-tree state or source hash alone establishes the producer of a firmware binary.

The TT3 captured `system` and TT1/Austin `system-5.5279.bin` are byte-identical (1,030,991 bytes, SHA-256 above). This is a `BYTE_IDENTICAL_DONOR`/analysis-input relationship for package bytes, not proof that Austin or TT3's active NOR contained those bytes. TJ Atlas III materials are `CROSS_FAMILY` and `COMPARATIVE_ONLY`: their TTBL structure can inform family-level context, while SoC-specific addresses, implementation and installed-NOR identity do not transfer to Tomi. Other-device catalogs are outside this reference.

## Preservation rules

The project’s established practice supports these rules:

- Preserve the untouched acquisition and its acquisition metadata separately from extracts, reconstructions, and modified candidates.
- Record byte count, cryptographic hash, source/context, method, and date for each acquired or generated specimen where available.
- Perform transformations on working copies; retain the primary specimen and record the exact input/output relationship.
- Treat byte-identical reconstruction as a reproducibility result, not a new primary acquisition.
- Never let a candidate number, filename, version string, operator report, or matching hash substitute for evidence of installation or execution.
- Do not overwrite the sole known-good specimen; keep rescue claims distinct from independently acquired rescue bytes.

## Preservation gaps and known unknowns

- No acquired live NOR/bootloader byte image; the `SYSTEM` update package cannot fill this gap.
- No single capture covers all nonvolatile Tomi storage; raw MMC does not include NOR/NGFFS.
- The complete raw MMC acquisition is locally integrity-verified, but exact source-state equivalence was not independently established because its later source hash was taken after a normal boot.
- No current installed `ttsystem` re-hash or current installed-user-space byte binding is available in the reviewed canon.
- `.tomi-rescue/ttsystem.good`, its sidecar, `.failed`, and staging files are not all independent acquired specimens.
- V003 installation/restore chronology and the relationship between reported boot outcomes and the later namespace acquisition remain unresolved; no physical reboot qualified the SFN correction.
- The live version string is not bound to a preserved kernel hash; candidate/source build provenance does not establish current execution.
- OpenTom is dirty and not an immutable source snapshot; exact source-to-image reproducibility is not established for the preserved candidates.
- Raw transcripts and physical timestamps are incomplete for some operator-reported deployment/recovery events.

## Tooling inventory

| Tool / set | Class | Provenance role and boundary |
|---|---|---|
| `TT3_Archival_Toolkit/TT3_Capture_Windows.ps1`, `TT3_Image_WSL.sh`, runbook | `REUSABLE` | Documented file capture and raw-device acquisition process; the resulting captures still require their own identity/provenance records. |
| `/mnt/d/Codex/TT3/toolchain/bin/ttimgextract`, `mkttimage-kernel-first` and companion scripts/source | `DOMAIN_SPECIFIC` | TomTom TTBL/container extraction and reconstruction; pinned/checksummed toolchain. Not a generic disk-imaging tool. |
| `roundtrip-tests/tt3-nochange-20260806T004411Z/validate_roundtrip.py`, manifests and comparison reports | `DOMAIN_SPECIFIC` | Reproducible validation for the historical TT3 `ttsystem` no-change round-trip; it does not establish live state. |
| `ddrescue`, acquisition transcript/map, local SHA-256 and file-manifest comparison | `REUSABLE` | Raw MMC capture/integrity record; ddrescue map and image identity are distinct artifacts. |
| TTBL bootloader scripts, static-analysis extracts and address maps under TT1 | `DOMAIN_SPECIFIC` | Analysis derivatives tied to the identified 5.5279 package; not live NOR acquisition. |
| Candidate-specific V002–V007 builders/validators and reports | `DOMAIN_SPECIFIC` | Constructed and validated candidate lineage; each report’s source and build boundary governs. They do not prove deployment. |
| V003 raw-sector/SFN examination and disposable VFAT tests | `ONE_OFF` | Bounded recovery investigation; not a reusable successful physical recovery procedure. See the recovery runbook. |

No scripts were moved or promoted as part of this migration.

## Canonical references and provenance

- [Evidence Index](../../evidence/INDEX.md) — high-value retrieval anchors.
- [Tomi device profile](../devices/tt3-tomi.md) — device and observed-state identity.
- [Tomi boot chain](../architecture/tomi-boot-chain.md) and [bootloader image reference](tomi-bootloader-image.md) — boot path and package analysis.
- [Tomi USB-role kernel reference](tomi-usb-role-kernel.md) — V002–V007 implementation and candidate findings.
- [Tomi disaster recovery](../runbooks/tomi-disaster-recovery.md) — operational recovery and safety boundaries.
- [Tomi file transfer](../runbooks/tomi-file-transfer.md) — transfer mechanics.
- [Lab environment](../environment/lab-environment.md) — path and source-tree context.

This ledger is a canonical synthesis of the identified indexed acquisitions and current LUCE references as reviewed on 2026-09-24. The external migration reconciliation record lists the discovery families, exact evidence anchors, and conflicts considered. No device, storage, or source-tree mutation occurred in preparing this document.
