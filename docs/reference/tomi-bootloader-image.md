# Tomi Bootloader Image Reference

## Scope and image identity

This reference distinguishes the bootloader **update package** found on Tomi's captured filesystem from a raw dump of the bootloader currently executing from nonvolatile boot storage. No live NOR bootloader dump was acquired in the cited work.

| Specimen | Size | SHA-256 | Provenance and use |
|---|---:|---|---|
| TT3 root-level `SYSTEM` | 1,030,991 bytes | `6d899cc700e25fc73e74487644458fe14984e3c26a508fc47d7f0d07f62c10da` | Extracted from `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/filesystem-copy/system`; byte-identical to the TT1/Austin analysis input. Filesystem capture (`COMPLETE.json`) explicitly says no whole-device raw image was made. |
| TT1/Austin `system-5.5279.bin` | 1,030,991 bytes | `6d899cc700e25fc73e74487644458fe14984e3c26a508fc47d7f0d07f62c10da` | `/mnt/d/Codex/TT1/system-5.5279.bin`; direct static-analysis specimen. Equality with TT3 `SYSTEM` is byte-for-byte, but it does not prove active NOR equality. |
| Decoded 5.5279 bootloader image | 258,408 bytes | `9e5072cc7ac2fa09a29182c2e66afae11c67d6625c93222cf8ed7907aa31274b` | `/mnt/d/Codex/TT1/tt5279-usb-boot-predicate-addendum/work/bootloader.decoded`; derived by inflating the embedded gzip member, runtime base `0x30b00000`. It is a transformed derivative, not a separately acquired device image. |
| Historical TT3 `ttsystem` | 4,212,715 bytes | `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf` | Captured 2026-08-04 at `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/filesystem-copy/ttsystem`; separate Linux TTBL container, not the bootloader. |
| Live-baseline TT3 `ttsystem` | 2,219,582 bytes | `14505f7b17f1037163ea448ec418924d1302f575db3a7c65ef416a42af4efc21` | Acquired 2026-08-31 at `/mnt/d/Codex/TT3/current-live-baseline/ttsystem`; distinct specimen, not yet mapped in the cited structural analysis. |

The TT3 filesystem capture's `bootloaderversion.txt` reports `BootloaderTarget=s3c24xx` and `Bootloaderversion=5.5279`; `ttgo.bif` independently reports numeric `55279`. These identify the reported bootloader version, not the bytes in live NOR.

## `SYSTEM` update-package layout

The following offsets are package-file offsets in the exact 1,030,991-byte `SYSTEM` specimen above. `decoded+...` offsets refer to the 258,408-byte inflated image, whose runtime base is `0x30b00000`. Ranges are half-open when explicitly marked “end-exclusive”; otherwise the inclusive ranges follow the source report.

| Package range | Length | Contents / interpretation | Evidence class |
|---|---:|---|---|
| `0x00000000–0x0000000b` | 12 bytes | TTBL magic/header metadata, including first-section size and load address. | Direct structure parse; confirmed. |
| `0x0000000c–0x0000200b` | `0x2000` (8,192) | Opaque, preserved non-entry payload data; no more specific role assigned. | Direct byte/cross-reference census; unresolved. |
| `0x0000200c–0x0000800b` | `0x6000` (24,576) | Zero fill. | Direct byte census. |
| `0x0000800c–0x0000b90b` | `0x3900` (14,592) | ARM reset/vector/bootstrap/inflater code and data, mapped at low NOR addresses `0x00008000–0x0000b8ff`. | Reset vector, coherent control flow, gzip checks and handoff; confirmed for this package. |
| `0x0000b90c–0x0002780d` | `0x1bf02` (114,434) | Gzip member named `bootloader`; payload-relative source starts at `0xb900`; expands to 258,408 bytes at `0x30b00000`. | Header and working inflate path; confirmed. |
| `0x0002780e–0x0003fffb` | `0x187ee` (100,334) | `0xff` padding/erased area. | Direct byte census. |
| `0x0003fffc–0x0004000b` | 16 bytes | Six-byte version text `5.5279` plus padding. The platform's `SYSTEM` probe reads six bytes at `0x3fffc`. | Direct bytes and code cross-reference; confirmed. |
| `0x0004000c–0x0004001b` | 16 bytes | First-section signature. First section payload length is `0x40000` bytes. | Direct structure parse; signature algorithm details are in the loader section below. |

The first section occupies file offsets `0x00000004–0x0004001b` including its header and signature; it is only the first of 19 nonempty sections in the full 1,030,991-byte (`0x0fbb4f`) update package. The full TTBL section directory, parsed directly from the package headers, is:

| Section | Header offset | Payload file range (end-exclusive) | Size (decimal) | Load address |
|---:|---:|---:|---:|---:|
| 1 | `0x00000004` | `0x0000000c–0x0004000c` | 262,144 | `0x31000000` |
| 2 | `0x0004001c` | `0x00040024–0x0005e97c` | 125,272 | `0x31700000` |
| 3 | `0x0005e98c` | `0x0005e994–0x000805ea` | 138,326 | `0x31200000` |
| 4 | `0x000805fa` | `0x00080602–0x00087beb` | 30,185 | `0x31228000` |
| 5 | `0x00087bfb` | `0x00087c03–0x0008fd40` | 33,085 | `0x31240000` |
| 6 | `0x0008fd50` | `0x0008fd58–0x0009afd8` | 45,696 | `0x31258000` |
| 7 | `0x0009afe8` | `0x0009aff0–0x000a33fc` | 33,804 | `0x31270000` |
| 8 | `0x000a340c` | `0x000a3414–0x000a951d` | 24,841 | `0x31288000` |
| 9 | `0x000a952d` | `0x000a9535–0x000b0d76` | 30,785 | `0x312a0000` |
| 10 | `0x000b0d86` | `0x000b0d8e–0x000bdd1f` | 53,137 | `0x312b8000` |
| 11 | `0x000bdd2f` | `0x000bdd37–0x000c78de` | 39,847 | `0x312d0000` |
| 12 | `0x000c78ee` | `0x000c78f6–0x000cb0d4` | 14,302 | `0x312e8000` |
| 13 | `0x000cb0e4` | `0x000cb0ec–0x000cee83` | 15,767 | `0x31300000` |
| 14 | `0x000cee93` | `0x000cee9b–0x000d89d0` | 39,733 | `0x31318000` |
| 15 | `0x000d89e0` | `0x000d89e8–0x000dd3b9` | 18,897 | `0x31330000` |
| 16 | `0x000dd3c9` | `0x000dd3d1–0x000e8051` | 44,160 | `0x31348000` |
| 17 | `0x000e8061` | `0x000e8069–0x000efe57` | 32,238 | `0x31360000` |
| 18 | `0x000efe67` | `0x000efe6f–0x000f5a6e` | 23,551 | `0x31378000` |
| 19 | `0x000f5a7e` | `0x000f5a86–0x000fbb2b` | 24,741 | `0x31180000` |

The zero-size terminator begins at `0x000fbb3b`; its entry is `0x31700000` and parameter is `0x30000000`. In this **bootloader update package**, the entry points at section 2. Do not confuse this update-package entry with the coincidentally equal entry recorded in the historical Linux `ttsystem` below. The first-section load/staging address `0x31000000` is also distinct from the bootstrap's observed low-NOR mapping and decoded-image base `0x30b00000`.

### Inflated-image map

| Decoded range | Runtime range | Contents | Confidence / boundary |
|---|---|---|---|
| `0x00000–0x28d67` | `0x30b00000–0x30b28d67` | Mixed ARM code, literal pools, inline strings, and platform records. | Code/data mix confirmed; not every byte is code. |
| `0x28d68–0x3cd3f` | `0x30b28d68–0x30b3cd3f` | Non-code assets and sparse tables. | Broad classification supported; exact substructure unresolved. |
| `0x3cd40–0x3dd87` | `0x30b3cd40–0x30b3dd87` | Blowfish constants/tables. | Confirmed by signature-key setup references. |
| `0x3dd88–0x3e323` | `0x30b3dd88–0x30b3e323` | Runtime/library strings and tables. | Strong string/reference evidence. |
| `0x3e324–0x3f167` | initialized data copied to `0x30008000–0x30008e43` | Writable initialized-data image. | Startup copy loop identified. |

The bootstrap at package payload-relative `0x8000` selects output base `0x30b00000`, uses gzip source `0x0000b900`, checks gzip magic `1f 8b 08`, inflates, then transfers control into the decoded image. These are findings about the analyzed `SYSTEM` package, not a direct observation of current Tomi NOR execution.

## Austin-configured boot-source and package loader findings

The matching image's static Austin/type-42 initialization configures `HSMOVINAND` and FAT16/32 as its grounded direct package source. Its 8.3 name set includes `DIAGSYS`, `SIGNAPPSIGN`, `SYSTEM`, `TTSYSTEM`, `LTSYSTEM`, and `CMDLINE.TXT`. The `SYSTEM` probe checks `TTBL`, requires a first-section size of `0x40000`, reads the six-byte version footer at `0x3fffc`, and selects among standard names. Exact product semantics for each probe result are unresolved.

The generic signed-TTBL loader:

1. Requires little-endian magic `0x4c425454` (bytes `TTBL`).
2. Reads section size and destination address; copies each payload to its declared address.
3. Computes MD5 and validates each 16-byte signature using the inline Blowfish key `d888d313ed83baad9cf41b50b343fadd` to decrypt the signature halves before comparison.
4. Reads entry and parameter values from the zero-size terminator and stores them at `0x30109fcc` and `0x30109fd0`.
5. Builds boot parameters and branches to the recorded entry (`bx r3` in the analyzed image).

For the analyzed `SYSTEM` update package, the TTBL tail entry is `0x31700000` and parameter is `0x30000000`. These are package-specific handoff values; do not transfer them to another `ttsystem` specimen or call the parameter's meaning known.

The same static analysis identifies USB Mass Storage as a device-side target, including CBW/CSW handling and SCSI read/write paths. It did not find a grounded USB-host/OHCI initialization or USB-storage boot source in this specimen. This is a bounded binary-analysis negative, not a proof about every bootloader revision.

## Historical TT3 `ttsystem` map

The historical 2026-08-04 TT3 `ttsystem` is an outer TTBL Linux-system container. Its no-change round trip verified the whole artifact byte-identical, including both payloads and the initramfs tree. Offsets below are file offsets; the payload identity and byte-exact validation are recorded in `/mnt/d/Codex/TT3/roundtrip-tests/tt3-nochange-20260806T004411Z/TTBL-STRUCTURE-COMPARISON.txt`.

| File range | Section / interpretation | Address or identity |
|---|---|---|
| `0x00000000–0x0012121b` | First section: 1,184,256-byte payload begins at `0x0c`; wrapper/decompressor at payload offsets `0x0000–0x35a3`, then gzip kernel at `0x35a4–0x12113d`; section signature begins at `0x12120c`. | Section address `0x31700000`; payload SHA-256 `6a6e585bb7653156d0cc5c661a214acb15667372441e88b497746d741d51a9b8`. |
| `0x0012121c–0x004047de` | Second section: 3,028,395-byte gzip initramfs payload begins at `0x121224`; payload ends exclusive at `0x4047cf`; signature follows. | Section address `0x31000000`; payload SHA-256 `e31842fa8300a38fef32bea34b8843fcb58ce361dffa0cee95c6c212518ef264`. |
| `0x004047cf–0x004047de` | Second-section 16-byte signature. | `893fb372c004f2122504038e468fa49d`. |
| `0x004047df–0x004047ea` | 12-byte trailing entry/parameter record; file length is `0x4047eb` bytes. | Raw bytes `000000000000703100000030`; parsed entry `0x31700000`, parameter `0x30000000`. |

First section signature at file offset `0x12120c`: `b5bc555b2606e468bcf4a3be5b1fdc3a`. The compressed kernel member is 1,170,330 bytes, SHA-256 `aa21b08897527e5440c1c5ec7fbd9359e8f909d30624a3d5d534337e77361946`. The 230-byte interval from the compressed kernel's end-exclusive offset `0x12113e` to the initramfs start `0x121224` spans the end of section-1 payload, its signature, and the next section header; the round-trip report classifies it as metadata/padding and does not assign internal semantics. The initramfs expands to 7,214,080 bytes, SHA-256 `0e4e240ddfb2acae51a6627a92b5e09b51f56f938d191b3dec1c698de30da1c8`.

This historical layout does **not** describe the separately acquired 2026-08-31 live-baseline `ttsystem` (2,219,582 bytes, SHA-256 above). Its format and internal map remain uncharacterized by the cited package.

## Extraction, reconstruction, and reproducibility

The preserved reproducibility set is `/mnt/d/Codex/TT3/roundtrip-tests/tt3-nochange-20260806T004411Z/`. `ROUNDTRIP-SUMMARY.txt` reports byte-identical original/rebuilt output, identical kernel and initramfs payloads/tree, identical TTBL structure, and no physical TT3 access. `validate_roundtrip.py`, `RUN-VALIDATION.sh`, and the toolchain under `/mnt/d/Codex/TT3/toolchain/` document the extraction/rebuild/verification path.

The matching bootloader specimen was inflated into `bootloader.decoded` for the static disassembly. The exact raw input and derived-image identities are listed above. The bounded disassembly excerpts are preserved under `/mnt/d/Codex/TT1/tt5279-disasm/`; the boot-storage predicate supplement is under `/mnt/d/Codex/TT1/tt5279-usb-boot-predicate-addendum/`.

## Comparative specimen: TJ Atlas III A1.0012

This is separate-family evidence, not a Tomi image or address map. The TJ ONE v8 source-storage image is `/mnt/d/Codex/TJ1/TJ_PE3398B05423_working.img` (1,027,604,480 bytes; SHA-256 `557a13ceab148b4e0f23922fccaa102f18b1ba362a603e8434356a6aea586e14`). Its extracted root-level `system` is 706,801 bytes, SHA-256 `2674420245fc0f4f5d89d6c5f8542e0dcd107b136010fd010babbf3f3971bdef`, and is a ten-section TTBL bootloader-update package. Section 0 contains a 256 KiB A1.0012/clist-216525 bootloader image; section 1 contains an ARM/CFI NOR updater; sections 2–9 hold board/display resources. The embedded bootloader payload expands to 176,400 bytes, SHA-256 `fa062c232de5eaeac69f8113bbc5f0331a8b48e3f1a6d547aad910fb35b24a08`, at `0xC0B00000`.

The Atlas specimen confirms that a second TomTom platform used TTBL sections, FAT fixed-name boot selection, TTBL package parsing/loading, preboot USB support, and Linux handoff construction. Its directly recovered fixed names include `DIAGSYS`, `SIGNAPPSIGN`, `SYSTEM`, `TTSYSTEM`, `LTSYSTEM`, and `CMDLINE.TXT`. These structural similarities are family-level evidence only: Atlas uses different hardware, addresses, memory placement, and board resources. Its exact USB admission predicate and signature acceptance semantics are unresolved; Austin 5.5279 offsets, addresses, and predicate are not transferable. The preserved report supports conclusions about the analyzed update-package specimen, not byte identity with TJ's installed NOR.

The related cross-family boot-drum investigation (`/mnt/d/Codex/TT3/tomtom-boot-drum-investigation-20260913/REPORT.md`) found the embedded 25,157-byte compressed sample byte-identical in the analyzed Austin 5.5279 and Atlas III 1.0012 loaders, while their audio hardware paths differ. That is a bounded shared-resource observation and does not establish loader identity or Tomi's active NOR bytes.

## Unresolved structures and limits

- The first 8,192 package bytes (`0x0c–0x200b`) remain opaque; no field names are assigned.
- The live bootloader's raw NOR image and byte identity are unavailable.
- The live-baseline `ttsystem` has not been structurally mapped here.
- `CMDLINE.TXT` use, version-probe policy labels, several padding/metadata fields, and the TTBL parameter's meaning remain unresolved.
- The broad asset/sparse-table interval is not decoded field by field.
- Direct code observations in the Austin-configured package are not by themselves a physical-runtime trace of Tomi.
- No safe bootloader write/recovery procedure is established by this forensic work.

## Related canonical reference

- [Tomi Boot Chain](../architecture/tomi-boot-chain.md) — stage ordering and Linux handoff relationship.
- [TT3 “Tomi” device profile](../devices/tt3-tomi.md) — device/platform identity and software specimen separation.
- [Evidence Index](../../evidence/INDEX.md) — selected external evidence locations.
