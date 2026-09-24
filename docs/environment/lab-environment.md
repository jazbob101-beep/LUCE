# TomTom Revival Lab Environment

## Purpose

This reference identifies the lab computers and execution environments, their durable roles, and where common source, build, and evidence files live. Choose the environment from the path and tool involved: Windows paths, WSL paths, MacBook-host paths, Bookworm-chroot paths, and Tomi target paths are not interchangeable.

## Environment map

```text
DEAN-PC Windows                         MacBook host (Debian 13)
  D:\Codex                                  /home/chris/data-share
      |                                      T: / \\macbook\data-share
      | /mnt/d/Codex from WSL                     |
      v                                           | host filesystem
DEAN-PC WSL                                   /opt/opentom-bookworm
  Codex / Linux tools / Git                       | chroot
  OpenTom work + ESP-IDF builds                    v
  /home/jazbob/                               Debian 12 Bookworm
                                                   /root/OpenTom
                                                   GCC 3.3.4 ARM

Tomi is the target device; it is not a general-purpose build host.
```

The diagram shows the lab's broad roles, not a file-transfer or device-access procedure. The TomiDock network architecture and its addresses are documented in [TomiDock Architecture](../architecture/tomidock.md); file movement is covered by [Tomi File Transfer](../runbooks/tomi-file-transfer.md).

## DEAN-PC / Windows

DEAN-PC is the Windows host. Its principal investigation and archival corpus is `D:\Codex`; from WSL the same Windows D: content is available at `/mnt/d/Codex`. Treat those as two path forms for one corpus, not independent copies. Use the Windows form in PowerShell and the POSIX form in WSL.

Windows-side work includes PowerShell operations that specifically target Windows paths and the documented `T:` share mapping. DEAN-PC's LAN address, when useful as orientation, is `192.168.1.117`; detailed network behavior belongs in the architecture document.

Do not assume a Windows-native command can consume a Linux path such as `/home/jazbob/...`, or that a WSL command should be given `D:\...` instead of its mounted equivalent.

## WSL / current Codex execution environment

Codex currently runs directly in WSL/Linux. Run ordinary shell, Git, `rg`, Linux analysis, and build commands in that environment with Linux paths. Do not wrap normal Codex commands in `wsl.exe -- bash -lc ...`; that pattern belongs to older Windows-side execution contexts and is superseded here.

### ESP-IDF and ESP32-S3 projects

The verified ESP-IDF installation is:

```text
/home/jazbob/esp/esp-idf-v5.5.5
```

Activate it in WSL with the installed export script when a task needs ESP-IDF:

```bash
source /home/jazbob/esp/esp-idf-v5.5.5/export.sh
```

The TomiDock ESP32-S3 project family is under:

```text
/home/jazbob/opentom/lab-work/esp32-s3/
```

Select the exact project named by the task. The family contains distinct production-lineage, diagnostic, and disposable candidates; a directory name or successful build does not establish which image is installed on the ESP. If serial access is part of an authorized task, attach the intended USB device to WSL and verify the actual device node before using it; `/dev/ttyUSB0` has been a common observed path, not a permanent assignment.

### OpenTom source and working projects

The WSL OpenTom checkout is:

```text
/home/jazbob/opentom
```

It is an upstream-derived working checkout (`origin` is `https://github.com/micahlt/OpenTom.git`), with historical/reference source under `src/`, including the S3C24xx kernel and OpenTom/initramfs skeletons. Local project work, including TomiDock projects, lives under `lab-work/`; generated builds, verification roots, staging inputs, and helper material may also exist in this checkout. These are different kinds of content, not a single pristine upstream source snapshot.

The checkout has local modifications and untracked project material. Inspect the exact working-tree status and exact project path for each task. Git `HEAD` alone does not identify untracked or modified inputs. Do not equate this source checkout, a disposable build copy, a preserved extracted snapshot, or a deployed Tomi/ESP binary with one another without matching hashes or other explicit identity evidence.

Historical Tomi firmware, captures, candidate packages, and investigation reports are organized separately in `/mnt/d/Codex/`, commonly under `TT3/`. Use the exact artifact/source identity established by the relevant report; do not substitute a similarly named sibling.

### LUCE

Codex's local LUCE clone is:

```text
/home/jazbob/LUCE
```

Its canonical remote is `jazbob101-beep/LUCE`; `main` is the canonical living-document branch. Use LUCE for current adjudicated documentation. OpenTom implementation work and bulky/raw evidence remain in their respective working/evidence locations unless a task explicitly migrates a document.

## MacBook

The MacBook host is currently verified as Debian GNU/Linux 13 (Trixie). It is the Tomi/OpenTom support and staging workstation, and hosts the legacy Debian Bookworm build chroot. The MacBook's `data-share` directory was present at verification time.

### Shared staging

These paths refer to the shared staging location:

```text
MacBook:       /home/chris/data-share  (also ~/data-share)
DEAN-PC:       T:  or  \\macbook\data-share\
```

The mapping is documented for moving staged files between the Windows host and MacBook. This environment reference records the relationship only; use [Tomi File Transfer](../runbooks/tomi-file-transfer.md) for actual transfer procedures. Do not assume that `T:` is a WSL mount.

### Bookworm chroot and legacy ARM toolchain

The MacBook host contains the existing Debian GNU/Linux 12 (Bookworm) OpenTom chroot at:

```text
/opt/opentom-bookworm
```

The established entry forms are:

```bash
sudo chroot /opt/opentom-bookworm /bin/bash
sudo chroot /opt/opentom-bookworm /bin/sh -c '...'
```

Inside the chroot, the OpenTom workspace and legacy target compiler are:

```text
/root/OpenTom
/root/OpenTom/gcc-3.3.4_glibc-2.3.2/bin/arm-linux-gcc
/root/OpenTom/gcc-3.3.4_glibc-2.3.2/bin/arm-linux-strip
```

The compiler was verified as GCC 3.3.4 inside the Bookworm chroot. Its host-side backing path is `/opt/opentom-bookworm/root/OpenTom/gcc-3.3.4_glibc-2.3.2/bin/arm-linux-gcc`; the parent `/opt/opentom-bookworm/root` is root-only, so ordinary MacBook users cannot traverse that path. Run legacy compiler commands in the chroot, not by treating the host backing path as the build context.

Use this preserved environment for legacy Tomi/OpenTom ARM builds when compatibility with its historical compiler, libraries, and build assumptions matters. The established compiler path is evidence, not a recommendation to rebuild every OpenTom component. Do not silently substitute a modern cross-compiler or a compiler copy in another environment. Stage experimental builds separately when the source/evidence gate requires it; do not build into a canonical or preserved source tree merely because it is present.

## Tomi target

Tomi is the embedded target, not a general build host. On-device work is limited to the runtime observation, testing, or deployment explicitly authorized for a task. Perform source inspection and builds on the appropriate workstation/environment; use the canonical device profile, runtime reference, architecture note, and runbook for Tomi-specific details. A source build does not prove that its output is currently installed on Tomi.

## Where commands should run

| Work | Environment |
|---|---|
| Codex shell, Git, Linux source inspection, local firmware analysis | DEAN-PC WSL, using `/home/jazbob/...` and `/mnt/{c,d}/...` paths |
| ESP32-S3 ESP-IDF build | DEAN-PC WSL with `/home/jazbob/esp/esp-idf-v5.5.5` activated and the exact project selected |
| Windows-only path/share operations | DEAN-PC Windows/PowerShell, using `D:\Codex` and the documented `T:` mapping |
| Legacy Tomi/OpenTom ARM target build requiring the pinned historical ABI/toolchain | MacBook Debian 12 Bookworm chroot, using `/root/OpenTom/...` paths |
| Staging/intermediary host work and MacBook-resident artifacts | MacBook Debian 13 host, using `/home/chris/...` or the chroot where appropriate |
| Target runtime inspection or authorized deployment | Tomi/device workflow, not a build environment |

When commands cross environments, label the machine and path context. A path that exists inside a chroot is not automatically readable by the unprivileged host account; a Windows drive path is not automatically a Linux path; and a source/build artifact is not an installed-image identity.

## Durable path reference

| Purpose | Path |
|---|---|
| Windows evidence/corpus | `D:\Codex` |
| Same corpus from WSL | `/mnt/d/Codex` |
| Current Codex Git documentation | `/home/jazbob/LUCE` |
| OpenTom upstream-derived WSL working checkout | `/home/jazbob/opentom` |
| TomiDock ESP32-S3 project family | `/home/jazbob/opentom/lab-work/esp32-s3/` |
| ESP-IDF 5.5.5 | `/home/jazbob/esp/esp-idf-v5.5.5` |
| MacBook shared staging | `/home/chris/data-share` / `~/data-share` |
| Windows view of staging share | `T:` / `\\macbook\data-share\` |
| Bookworm chroot on MacBook host | `/opt/opentom-bookworm` |
| OpenTom workspace inside chroot | `/root/OpenTom` |
| Historical ARM compiler inside chroot | `/root/OpenTom/gcc-3.3.4_glibc-2.3.2/bin/arm-linux-gcc` |

## Canonical references

- [Current project state](../../CURRENT_STATE.md)
- [TomiDock architecture](../architecture/tomidock.md)
- [TT3 “Tomi” device profile](../devices/tt3-tomi.md)
- [Tomi Runtime ABI](../reference/tomi-runtime-abi.md)
- [Tomi file-transfer runbook](../runbooks/tomi-file-transfer.md)

## Provenance

Current DEAN-PC/WSL and MacBook/chroot roles and established paths are reconciled from the 2026-09-24 TomiDock handoff, the live WSL paths, the local OpenTom/ESP-IDF checkouts, and a read-only MacBook/chroot verification. The legacy build-context distinction is corroborated by the V002 OpenTom build-validation report. The Windows share relationship and file movement are owned in detail by the Tomi file-transfer runbook. Exact source and evidence paths, historical wrapper references, and limits of verification are recorded in the external reconciliation report for this migration.
