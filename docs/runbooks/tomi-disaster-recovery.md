# Tomi Disaster Recovery

## Scope and terminology

This is a TT3 “Tomi”-specific operational map of recovery paths established by the project. **“Boot-time disaster recovery mode” is not a native TomTom bootloader mode established by this evidence.** It refers to a project-added Linux initramfs rescue trigger and restore transaction, which runs only after a Linux kernel has started and early `/etc/rc` is executing. A distinct bootloader USB Mass Storage (MSC) path can expose storage before Linux handoff; it is not the same recovery mechanism.

This runbook does not authorize a repair, reboot, flash, or media write. It gives the safest known escalation and marks unvalidated paths. For a live incident, first determine **what still boots**. Preserve the observed display, USB appearance, and logs; do not infer a precise fault from a red X, absent desktop, or absent network alone.

For bootloader stages and package boundaries, see [Tomi Boot Chain](../architecture/tomi-boot-chain.md) and [Tomi Bootloader Image](../reference/tomi-bootloader-image.md). For routine verified file movement, use [Tomi File Transfer](tomi-file-transfer.md). USB role is mutable state, and the forced-host sysfs write is prohibited by [Tomi USB Role / Kernel](../reference/tomi-usb-role-kernel.md).

## Recovery model: identify the surviving layer

The layer labels below are a decision aid, not Tomi firmware terminology.

| What is observed | Likely boundary (not a diagnosis) | Least-invasive established path | Confidence / limitation |
|---|---|---|---|
| Linux/OpenTom desktop works; one app/service fails | Linux and normal userspace are available | If shell access works, preserve diagnostics and repair only that component using verified transfer and rollback practice. | Path is feasible; no universal repair transaction for every service is established. |
| Linux reaches early boot/`/etc/rc`, but Gadget/network or later startup is broken | Kernel/initramfs are running; later setup may be faulty | Project rescue gesture can request a known-good `ttsystem` restore, then retain maintenance mode. | Physical V002 rescue outcome was operator-reported successful. Requires intact trigger, early `rc`, MMC/FAT mount, restore binary and verified on-card backup. |
| Bootloader failure display; Linux does not appear; USB cable to a host changes to bootloader MSC | Bootloader still appears to run and exposes its storage service | Host can inspect the exposed storage. A specific V003 filename-namespace repair was performed, but its clean-fsck/reboot gate failed; do not treat it as a ready-to-repeat repair recipe. | MSC transition/operator-visible device was reported; failure-loop-to-MSC path is statically mapped. V003 repaired state was not reboot-validated. |
| No Linux, no bootloader MSC, or no evidence bootloader remains responsive | Failure may be before/within bootloader, storage, or power path | No TT3 bootloader-write or hardware recovery procedure is established here. Stop before attempting writes; preserve exact symptoms and seek a separately reviewed path. | UNRESOLVED. No NOR flashing instructions. |

The project-added initramfs rescue cannot repair a failure that prevents kernel/initramfs handoff, prevents early `/etc/rc` from reaching the trigger, or prevents the main storage from mounting. It is not an automatic watchdog rollback: it waits for an intentional physical gesture.

## Proven project Linux rescue path

### Trigger and stage

The preserved initramfs contains `/bin/tomi-rescue-trigger` (18,772 bytes, SHA-256 `f6d74246cfc69bf7195df40406e24f11ab0acc59d4c765ef8303c31ecfc0c7ed`) and `/bin/tomi-rescue-restore` (20,592 bytes, SHA-256 `8c82d86b4172af5b32b50a452eda524bb668b443f414f49a17996b89819051d6`). These are project-specific ARM binaries, not bootloader features. The trigger's embedded diagnostics identify raw GPF0 sampling and a production 3,000 ms observation window for a three-tap sequence. It waits for a debounced released state before arming; the initial power-on hold is not itself the rescue sequence. The trigger can still detect the input if its framebuffer presentation is unavailable. Exact on-device button timing/log capture for the V002 rescue is not preserved separately; gesture semantics are binary-observed, while the successful V002 result is operator-reported.

In the preserved `/etc/rc`, `initial_mount` and the MMC settling delay precede the trigger. Storage is mounted after the trigger returns. A successful trigger result is then passed to `/bin/tomi-rescue-restore --live`; on restore success, `rc` syncs twice and calls `force_reboot`. A restore error enters maintenance mode rather than continuing normal OpenTom startup. Missing trigger or its reported error does **not** reliably force recovery: normal boot continues after the warning path.

### What the restore transaction does

Static evidence from the exact restore binary and preserved `/etc/rc` shows this intended transaction:

1. Check `/mnt/sdcard/.tomi-rescue/ttsystem.good` against `ttsystem.good.sha256` and inspect the active `/mnt/sdcard/ttsystem`.
2. Preserve the active image as `.tomi-rescue/ttsystem.failed` through a temporary `.failed.new` file, with write/hash checks.
3. Stage the verified good image as `.tomi-rescue/ttsystem.stage` and verify it.
4. Create the persistent `.tomi-rescue/RESCUE` marker, then rename the staged image to the active `ttsystem` path and verify the committed hash.
5. On success, early `rc` reboots; the next boot sees `RESCUE`, starts USB Ethernet/DHCP/Telnet, and suppresses `/mnt/sdcard/opentom/start.sh`.

The V002 bench report says the failed V002 image was retained as `.tomi-rescue/ttsystem.failed`, the exact pre-TomiDock baseline was restored, and the maintenance marker was removed after recovery. This is the strongest **BENCH_PROVEN (operator-reported)** recovery outcome: a broken USB-Gadget role did not prevent Linux desktop boot, and the physical rescue restored the prior known-good system. It is not evidence that the same method works when Linux cannot reach early `rc`.

The marker provides a persistent minimal maintenance path, not a general repair shell independent of kernel/Gadget health. Do not remove it merely to return to normal startup; no universal, current marker-clear procedure is established in this runbook. Confirm the restored image and follow a separately verified operator procedure before changing maintenance state.

### Preconditions and limits

This path depends on: bootloader selecting and loading a Linux image; kernel/initramfs execution; functioning early `/etc/rc`; trigger and restore binaries; main MMC/VFAT storage mounting at `/mnt/sdcard`; intact `.tomi-rescue` files and checksum sidecar; and enough functioning early USB-Gadget support for maintenance Telnet if needed. It does not replace the bootloader, repair internal NOR, repair a damaged card/filesystem, or guarantee a usable Gadget on every candidate kernel.

The transaction restores file bytes, but **does not inherently repair FAT directory-name metadata**. VFAT rename-over-existing can preserve an existing destination short-name/case representation. A known V003 failure involved a Linux-visible `ttsystem` whose dummy SFN was skipped by the bootloader; copying the correct image bytes through the rescue helper would not be a proven cure for that name defect.

## Bootloader MSC and storage-level recovery boundary

The matching Austin/S3C2412 5.5279 package statically supports a pre-handoff USB MSC service. A Tomi operator reported an indefinite failure display that changed to a disk-drive graphic and MSC enumeration after connecting USB to a host, without a Linux rescue restore between those states. The mapped code path shows the bootloader failure loop can enter its MSC path before Linux handoff; the exact red-X artwork is not a unique error code. The static USB predicate requires a host-driven USB reset latch and a clear storage-inhibit value; raw VBUS alone is not sufficient.

This can make the storage accessible to a host when Linux does not boot. In the V003 namespace investigation, the host obtained stable raw storage evidence and Linux VFAT move/copy operations created a genuine uppercase `TTSYSTEM` SFN containing the exact 2,219,582-byte baseline. However, the required read-only `fsck.fat -n` gate exited 1 on unrelated duplicate entries/free-space inconsistencies, and **no physical reboot followed**. That result is a **PARTIAL RECOVERY / IMAGE-VALIDATED STORAGE CHANGE**, not proof of a recovered boot or a general write recipe. Preserve that distinction; do not run filesystem repair mode or repeat rename/copy steps from this reference.

USB MSC is not “boot from a USB stick.” The analyzed selector reads the device's MMC/FAT storage. No project-proven alternate SD boot, USB-host boot, `DIAGSYS`/`LTSYSTEM` rescue image selection, or key-held bootloader alternate has been established. See the boot-chain reference instead of guessing from file-name strings.

## Known image identities — keep specimens separate

| Specimen | Identity | Recovery relevance |
|---|---|---|
| Current/live-baseline `ttsystem` | 2,219,582 bytes; SHA-256 `14505f7b17f1037163ea448ec418924d1302f575db3a7c65ef416a42af4efc21` | Exact baseline reported restored after V002. Also the bytes found in the later V003 storage acquisition and recreated under a valid `TTSYSTEM` SFN. It is not the 2026-08 historical image. |
| Historical 2026-08-04 captured `ttsystem` | 4,212,715 bytes; SHA-256 `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf` | Different specimen/kernel lineage. Not identified as the rescue image used in the V002 restore. |
| V002 USB-authority candidate | 2,228,840 bytes; SHA-256 `8755e8243778bcbc2e8c4820c2677f6dbc5eccf84c5d4e90995f4b7fd551a0c4` | Offline validated; operator reports full desktop boot but missing Gadget and successful physical rescue back to the live baseline. Candidate, not itself a recovery image. |
| V003 diagnostic candidate | 2,231,621 bytes; SHA-256 `cc7173c72bcbe5974940ac95685c9ff128807ea9ceed41045e7c10c63dc28d02` | Operator reported red-X failure. Later storage evidence found this byte sequence under a longer alternate filename and the baseline under `ttsystem`; exact install/restore chronology remains unresolved. |
| Root-level `SYSTEM` 5.5279 package | 1,030,991 bytes; SHA-256 `6d899cc700e25fc73e74487644458fe14984e3c26a508fc47d7f0d07f62c10da` | Bootloader update-package specimen, not live NOR bytes and not a Linux `ttsystem`. No operational NOR write procedure is established. |

Image hashes identify these preserved artifacts; they do not prove which bytes are currently installed or that any package is safe to deploy. The `.tomi-rescue/ttsystem.good` copy and its on-device sidecar were not separately acquired as independent current artifacts in the reviewed evidence.

## Failure classification and safe escalation

1. **Normal Linux boot, component failure:** retain logs and use the existing shell/transfer path. Avoid replacing the whole `ttsystem` for an isolated app/service issue. Follow the file-transfer runbook; verify staged bytes before promotion.
2. **Linux boots, USB/network path broken:** distinguish “desktop works, Gadget absent” from total boot failure. The V002 rescue is evidence that early Linux rescue can restore the prior image even when Gadget does not enumerate. Use only the known project gesture and only when the card/backup prerequisites are confirmed.
3. **Linux has not reached userspace, but bootloader MSC appears:** record the transition and preserve storage read-only first. The V003 host-side namespace correction is not a reboot-qualified repair; a new writable operation requires exact identity/ownership gates and a specific approved repair plan.
4. **No evidence of Linux or MSC:** stop. There is no established Tomi external rescue kernel or safe bootloader/NOR writer in this corpus. Escalation is evidence collection and specialist review, not speculative flashing.

Before replacing any system image, verify the exact specimen, destination, byte count, SHA-256, rollback copy, name/short-name behavior, and reboot gate. Do not overwrite the only known-good copy. Transfer mechanics and safe temporary staging are documented in [Tomi File Transfer](tomi-file-transfer.md); do not improvise rename choreography from this runbook.

## Unsafe, unproven, and out-of-scope paths

- **Never** force HOST with `echo host > /sys/devices/platform/tomtomgo-usbmode/mode`; it bypasses role arbitration and caused a documented Oops/`rc=139` incident. It is not recovery.
- Do not flash or replace `SYSTEM`, NOR, or bootloader data. Static updater code and a root-level update package do not establish a safe write procedure.
- Do not call historical `ttsystem`, the later live baseline, V002, and V003 interchangeable. Do not designate V002/V003 as “rescue images.”
- Do not infer that holding a button selects `DIAGSYS`, `LTSYSTEM`, or another loader image. Only the project-added early Linux three-tap trigger is evidenced as a Tomi rescue gesture.
- Do not use USB MSC availability as proof the filesystem is healthy, read-only, or Linux will boot. The bootloader MSC path supports host writes; protect and verify the exact device before any authorized storage operation.
- Do not use `fsck` repair mode, raw-sector edits, card formatting/reimaging, or a guessed `mv`/rename sequence as an emergency shortcut.
- TT1/Austin and TJ/Atlas III recovery findings are comparative only; their hardware, bootloader versions, memory maps, and rescue procedures are not transferable to Tomi.

## Known gaps and canonical references

Still unresolved: independent acquisition of the live `.tomi-rescue` directory and sidecar; byte-bound identity of its `ttsystem.good`; complete source and target-side execution trace for the restore helper; the undocumented marker-clear procedure; a clean-fsck, reboot-validated recovery for the V003 SFN defect; exact historical V003 installation/restore commands; and any recovery path for bootloader/NOR failure. These gaps preclude claiming a general brick-recovery guarantee.

- [Tomi Boot Chain](../architecture/tomi-boot-chain.md) — boot stages and Linux handoff boundary.
- [Tomi Bootloader Image](../reference/tomi-bootloader-image.md) — distinct package/image specimens and format evidence.
- [Tomi USB Role / Kernel](../reference/tomi-usb-role-kernel.md) — USB role meanings and unsafe forced-host boundary.
- [Tomi File Transfer](tomi-file-transfer.md) — current verified transfer workflow.
- [Current State](../../CURRENT_STATE.md) — current operational baseline, not a historical image ledger.

## Provenance

Primary recovery evidence is the exact pre-TomiDock baseline/initramfs and V002 rescue report in `/mnt/d/Codex/PPP-investigation/tomidock-boot-host-v001`; V002 operator-reported bench recovery in `/mnt/d/Codex/usbmode-authoritative-state-v003-blackbox-20260902/REPORT.md`; the V003 boot failure and loader audit in `/mnt/d/Codex/usbmode-v003-redx-forensic-20260902`; and the live storage/namespace continuation in `/mnt/d/Codex/usbmode-v003-redx-continuation-20260902`. See the external reconciliation report for the full investigation inventory, identities, success/partial/failure classifications, and index candidates.
