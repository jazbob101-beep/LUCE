# Tomi Linux USB Role Kernel Reference

## Scope

This reference records what is established about the Linux-era Tomi USB-role
arbiter, its S3C24xx implementation, its historical sysfs interface, and the
later authoritative-role candidate lineage. It does not define the complete
TomiDock topology or service lifecycle; see [TomiDock architecture](../architecture/tomidock.md).
Pre-Linux USB behavior belongs to the [boot chain](../architecture/tomi-boot-chain.md)
and [bootloader image reference](tomi-bootloader-image.md). General Tomi identity
is in the [device profile](../devices/tt3-tomi.md).

The findings below explicitly distinguish:

- **Live observation** — behavior or identity reported from the running Tomi.
- **Exact-image finding** — analysis tied to a cryptographically identified
  `ttsystem` or candidate image.
- **Source finding** — behavior in identified OpenTom source files; not proof
  that those exact files produced the live kernel.
- **Candidate/build finding** — a patch or image compiled/validated offline;
  not proof of deployment or runtime behavior.
- **Bench observation** — operator-run behavior on the named candidate/specimen.

## Evidence and kernel specimens

The relevant source audit used the OpenTom repository at Git HEAD
`eb3d2037315bde5da2875b38cba4efba9d47eac7`, working tree
`/home/jazbob/opentom`. The repository/tree has unrelated local modifications
and untracked build inputs. The relevant audited working files are the original
source identities recorded by the audit, but this source tree is not proven to
be the source of any preserved Tomi image or the current live kernel.

| Specimen | Identity | What it establishes |
|---|---|---|
| Historical TT3 `ttsystem` capture (2026-08-04) | 4,212,715 bytes; SHA-256 `ca37e2e7805f3a95530f0fb35e3d6edcb1f2c7b0fd53edf179e1a368745bacdf`; embedded Linux `2.6.13-tt567329`, build `#1`, 2010-08-24 | Exact historical bundle identity; not the currently running kernel. |
| Live-baseline `ttsystem` file (acquired 2026-08-31) | 2,219,582 bytes; SHA-256 `14505f7b17f1037163ea448ec418924d1302f575db3a7c65ef416a42af4efc21` | Exact preserved file identity. It does not bind the later live `uname` observation to this specimen. |
| Live kernel observation (reported 2026-09-24) | `Linux (none) 2.6.13 #9 Wed Sep 9 13:11:52 UTC 2026 armv5tejl GNU/Linux` | A live version string, not a cryptographic kernel identity. |
| V002 candidate image | 2,228,840 bytes; SHA-256 `8755e8243778bcbc2e8c4820c2677f6dbc5eccf84c5d4e90995f4b7fd551a0c4` | Reconstructed candidate, offline image validation passed; later operator report says desktop booted but Gadget did not enumerate. |
| V003 diagnostic candidate | 2,231,621 bytes; SHA-256 `cc7173c72bcbe5974940ac95685c9ff128807ea9ceed41045e7c10c63dc28d02` | Instrumented V002-lineage candidate; kernel and modules were preserved while startup logging was added. Not a role-policy fix. |
| V004 logger-compatibility candidate | 2,231,793 bytes; SHA-256 `45ad091ce4a5ad4c500f37a9bf601b9719019e706bd31eec3cc825463453aeb7` | Corrected exact-shell logging helper; USB policy remained that of the diagnostic predecessor. |
| V005 input-to-worker trace candidate | 2,233,931 bytes; SHA-256 `1398b7cf9016ccd32ac7674b2735ac31c0a168a6843da19e09ef61ecef124e52` | Diagnostic trace candidate; not a production image. |
| V006 notifier-repair candidate | 2,234,015 bytes; SHA-256 `066debd33caf8425690d5ef25e5ac20fee9e38733e2f60c30b58f77fb946f4d1` | Built candidate removes the confirmed destructive buspower notifier reset while retaining V005 diagnostics. |
| V007 production-lineage candidate | 2,228,744 bytes; SHA-256 `344a8b733b8cdef3c58705a2a7e8039e01024db547a71773a7d8cec20cdd2825`; zImage SHA-256 `b582d403d76da2da26bcc3d50b3161717e39ee4fc517c1a0bc2a376bbcab35f1` | Cleaned production candidate based on V002 role authority plus V006 notifier repair, with lab tracing removed. Built and statically validated; not deployed and no V007 Tomi run is established. |

V002 compiled `vmlinux` was 3,038,203 bytes, SHA-256
`da810529e7678069dd0169dc6e9e87b62a3ac6d247ac1b68b54741836ea34a46`.
V007 compiled `vmlinux` was SHA-256
`2827cee96f367f3518b2ac28350d0a1f8fa6d6d1d9e7de8b6382b1e54daff882`.
These identify build outputs, not the currently running kernel.

## USB role model

### Legacy source model (Era A)

The audited TomTom source represents the role with seven enum values:

| State | Source-level role meaning / caveat |
|---|---|
| `INITIAL` | Initial/quiescent setup before the late-init transition to `IDLE`. |
| `IDLE` | Arbiter has no selected operational role; legacy independent drivers could nevertheless create a contradictory effective controller state. |
| `DEVICE_DETECT` | Temporary peripheral-side detection: device pull-up/reset IRQ and a two-second timeout. |
| `DEVICE` | A host-generated USB reset was detected during `DEVICE_DETECT`. Historical comments expect bootloader device operation; legacy Linux UDC registration was not enforced as a lease of this state. |
| `HOST_DETECT` | Host clocks/pad routing are enabled while the implementation checks for a recognized USB child. |
| `HOST` | Legacy host state after a recognized child; the state and provider lifecycle were not a single enforced authority. |
| `CLA` | Two-second host-detection interval expired without a child. Legacy code retained host setup; its `HOST_DEVICE_ADDED` branch was a TODO/no-op. |

Normal source sequence for host-capable Austin hardware is:

```text
INITIAL --late init--> IDLE
IDLE --logical power on--> DEVICE_DETECT
DEVICE_DETECT --UDC reset--> DEVICE
DEVICE_DETECT --2 s without reset--> HOST_DETECT
HOST_DETECT --recognized child--> HOST
HOST_DETECT --2 s without child--> CLA
```

This is not a complete transition diagram: the event handler includes
state-specific power-off, child-removal, and suspend cases, and operations may
fail before the state variable is committed. In the legacy HOST path, Austin
can defer power-off handling until child removal. The source-level state is
therefore role-arbitration state, not an instantaneous physical cable,
controller-health, enumeration, or network-service indicator.

### Later authoritative contract (Era B)

The V001–V007 candidate family tightened the intended meaning without changing
the seven-state ABI. One serialized authority admits and revokes the shared
connector providers before publishing a stable role. In that contract,
`DEVICE_DETECT` temporarily owns arbitration resources, `DEVICE` gates UDC use,
`HOST_DETECT` classifies initial host topology, `HOST` requires a relevant
published direct-root child, and `CLA` retains an eligible host provider with
zero such children while continuing topology observation. These are candidate
contract semantics, not a claim that the live device is running V007.

Even this stronger state is not physical-cable truth or service health. Power
sense, cached buspower state, queued role input, committed state, controller
ownership, USB publication, and downstream IP/service state are distinct
observations. Compare correlated observations and eventual reconciliation;
do not treat separately sampled values as atomic.

## Kernel implementation

### Platform driver and source identity

The platform driver name is `tomtomgo-usbmode`; source registers it on the
platform bus. Principal files in the audited Linux 2.6.13 tree are:

- `src/linux-s3c24xx/drivers/barcelona/usbmode/usbmode.c` — generic state,
  event handling, sysfs attribute, listeners, and PM callbacks. SHA-256
  `51a0831063085191372fca5253dd108c59207aae32780ba6a20f51690902f48d`.
- `src/linux-s3c24xx/drivers/barcelona/usbmode/s3c24xx_usbmode.c` — S3C24xx
  operations, timer/IRQ paths, GPIO and controller routing. SHA-256
  `e3e6b2aaee0b3478060c7fd6c885f610d2660d6944f8ef5a718c8885d82f9614`.
- `src/linux-s3c24xx/include/barcelona/usbmode.h` — enum, event and API
  declarations; original SHA-256
  `869e2503d528014e7ada9c078709bff2ec60996fa89432a1340d9224f30f6866`.
- `src/linux-s3c24xx/drivers/barcelona/buspower/buspower.c` — sampled external
  power input and notifier producer. SHA-256
  `af3cd5beeef12f7f98a13cd7d9bd6fa13234a7f82fedbc2e43e4a2abfb08dce6`.

The three source hashes above are the audited original-file identities, not
assertions of exact correspondence to the historical or current live image.
The standalone kernel audit reports S3C2412, built-in TomTom buspower/usbmode,
modular OHCI and S3C24xx host glue in its inspected configuration. That
configuration itself is not the live device configuration.

### Sysfs interface

The platform device exposes `/sys/devices/platform/tomtomgo-usbmode/mode` in
the live observation and in source. `usbmode_sysfs_read_mode()` formats the
current enum as one of the seven `USB_STATE_*` strings. It does not query the
VBUS pin, OHCI port, USB child topology, UDC binding, or network health. The
legacy show path is a direct enum read, not a synchronized multi-layer
snapshot.

The legacy attribute is declared `S_IWUGO | S_IRUGO` and has a store handler.
That handler is not a safe role-control API:

- it accepts any buffer beginning with the four bytes `host` (so shell input
  `host\n` passes);
- from states other than `HOST_DETECT` and `CLA`, it calls the
  `device_detect_to_host_detect` backend helper directly, bypassing normal
  event/commit/listener paths;
- for `HOST_DETECT` and `CLA`, it writes `USB_STATE_HOST` before the backend
  helper succeeds;
- helper failures and malformed writes can still return the supplied byte
  count, so userspace may see false success.

Never recommend or execute:

```sh
echo host > /sys/devices/platform/tomtomgo-usbmode/mode
```

It bypasses normal arbitration and has caused a documented Tomi bench kernel
Oops/`rc=139` incident. The source has a concrete unsafe cleanup path below;
the incident is strongly source-compatible with it, but is not bound to a
cryptographically identified live kernel or exact faulting instruction.

### Normal power, device, and host paths

In the legacy backend, `buspower` samples Austin `USB_HOST_DETECT` about every
200 ms and notifies the role code when the sampled/cached logical value changes
or a re-notification is requested. Austin defines this as inverted GPF1,
shared with `ACPWR` and `IGNITION`: raw low reads logical power-on and raw high
logical power-off. It is a sensed input, not a voltmeter or proof of the peer's
USB role. A software RDS/TMC override can report logical off without reading
the pin.

`IDLE -> DEVICE_DETECT` runs `s3c24xx_idle_to_device_detect()` and
`device_detect_enter()`: device-side setup, separate Austin GPF7
`USB_PULL_EN`, a 100 ms stabilization delay, UDC interrupt status clear,
`IRQ_USBD` acquisition, USB-device clock use, and a two-second timer. A
recognized UDC reset drives `DEVICE_DETECT -> DEVICE`; timeout tears down
detection and selects host pad/clock setup, then `HOST_DETECT`.

Legacy `host_detect_timeout()` checks the USB bus for recognized devices.
Child presence can lead to `HOST`; otherwise timeout leads to `CLA` while
host-side resources remain. The old CLA add event is unimplemented, so a late
child could remain unreflected in the enum. V007's authority path instead
reconciles published child topology continuously: HOST may become CLA at zero
children and CLA may return to HOST after a configured child is published.
Publication is later than physical attach/port CCS; unsuccessful enumeration
does not satisfy the HOST predicate, and CLA may persist if no child is
published.

### Ownership, synchronization, and error behavior

Legacy events are accumulated and processed by a worker, but the sysfs store
mutates state/backend separately; no common transition lock/commit path was
found for those competing writers. The generic transition handler generally
calls the backend first and publishes the enum after success. The manual sysfs
path violates that ordering. Suspend/resume also directly manipulate/reset
state outside the common transition notification path.

The audited legacy detector backend allocates implementation storage with
non-zeroing `kmalloc()`. Its timer is initialized only after IRQ and clock
acquisition; `device_detect_exit()` nevertheless calls `del_timer_sync()` and
`free_irq()` unconditionally. A manual host write from IDLE can call the
device-detect-to-host path without owning a successfully initialized detector
timer/IRQ. The source audit says this exactly permits the observed
uninitialized-timer Oops. This is the best-supported mechanism, not a proven
instruction-level reconstruction of the historical crash.

The V001–V007 architecture candidate addresses the semantic design problem
with a dedicated serialized role worker, explicit resource ownership and
transactional entry/exit, provider admission/revocation, worker-driven state
commit, role/error diagnostics, and USB-core child-topology reconciliation.
V005 tracing then exposed a distinct producer-lifecycle defect: `buspower_probe`
reset `buspower_notifier_list` after a consumer could register. V006 removed
that reset; V007 retained the repair and removed experimental tracing. V007
passed offline ARM build, ABI, image, archive, and rescue validation, but its
image is not established as deployed or run on Tomi.

## Hardware and controller relationship

The Austin/S3C2412 working source maps:

- `USB_HOST_DETECT`, `ACPWR`, and `IGNITION` to inverted GPF1 input;
- `USB_PULL_EN` to separate GPF7 output;
- USB role/pad selection and suspend controls through S3C2412/S3C24xx
  `MISCCR` handling;
- host-controller platform resource base `S3C2410_PA_USBHOST = 0x49000000`
  and `IRQ_USBH`; lightweight device-detect/UDC uses `IRQ_USBD`.

These are source-defined addresses/signals in the audited Austin tree, not
live MMIO reads or board-schematic validation. No exact current live register
snapshot was identified. Generic OHCI and the S3C24xx host glue coexist with
TomTom-specific clocks, pad routing, VBUS sensing, role selection, and PM.
The exact register bit effects remain controller/tree-specific; no TJ/Atlas
address or behavior is imported here.

## USB suspend and Linux-side trigger boundary

ESP-side TinyUSB `SUSPEND` is not a Tomi Linux role state and is not, by
itself, evidence of physical cable removal. In the audited TinyUSB/DWC2 path,
approximately 3 ms of bus idle invokes the suspend callback while the device
remains connected and configured; TinyUSB marks it suspended and
`tud_ready()` becomes false. A resume/wakeup path invokes resume, not MOUNT.
MOUNT is associated with a nonzero USB `SET_CONFIGURATION` after
configuration has been cleared. These stack semantics explain event meaning,
but an application MOUNT log without setup/controller traces does not identify
the preceding physical or controller event.

The bounded stock-Linux source audit identified several intentional ways the
Tomi host stack could suspend or be restarted: explicit child-device or
root-hub runtime-PM writes; system suspend and its unwind; USB role/provider
teardown and restart; and OHCI root-hub autosuspend when its ports satisfy the
kernel's suspend/remote-wakeup conditions. An already-connected, nonsuspended
ECM child vetoes the audited generic OHCI root-hub autosuspend condition.
Interface-only usbnet runtime PM can stop interface traffic without stopping
USB Start-of-Frame traffic. The platform OHCI suspend hook examined in the
source was commented out, although resume code can restart the controller.
No stock writer to the relevant child, root-hub, or OHCI `power/state` control
was found in the bounded search; this is not proof that no runtime actor or
out-of-tree component can trigger the paths. No direct charging or battery
policy change was identified in these suspend paths. The triggering actor for
the historical observations remains unknown.

## RUN-2 black-box evidence boundary

The preserved V004 RUN-2 raw archive is SHA-256
`dae20f7af9f8c08526a7f715182d7f87268b9e88de760fc48d846ede5315722f`.
Its cumulative dmesg snapshots retain the boot prefix through the final
snapshot at uptime 311.35 seconds, with no observed truncation or ring
rollover. Within that captured dmesg interval, the archive contains no
explicit `authority entry failed` message, logged `last_error` change, or
additional committed authority transition. This excludes those *logged* events
only for the covered interval; it cannot exclude a silent/transient fault or
behavior after the final dmesg snapshot. The 229 polls span uptime 11.12 to
316.02 seconds and consistently sample IDLE with `usb0` and `g_ether` absent
and provider modules present. They leave a 4.67-second poll-only tail without
cumulative dmesg coverage. Polling also cannot prove absence of a transient
between samples.

The archive does not record a physical event timestamp or VBUS values and
does not establish successful authority admission. It narrows the explicit
logged-failure alternative (E) only during dmesg coverage; it does not resolve
input/producer failure (A) versus queue/reconciliation failure (B). Preserve
that boundary rather than treating stable sampled state as proof of every
intermediate transition.

## Boot initialization and userspace relationship

The generic driver registers as a platform driver; late init performs
`initial_to_idle()` and commits `IDLE` on success. In the later role-aware
boot integration, userspace treats the kernel role as a gate for starting
device-side Gadget services and starts TomiDock networking via the already
documented service path. The live system has been reported role-aware and
`USB_STATE_HOST` at one observation. Exact installed `/etc/rc`, kernel, and
service binary hashes are not bound by that report. Operational service and
TomiDock details remain owned by [current state](../../CURRENT_STATE.md) and
[TomiDock architecture](../architecture/tomidock.md).

ESP-side suspend/MOUNT events are not Linux role states and do not prove a
physical connector event. Historical application-level lifecycle anomalies
are maintained in the TomiDock reference; they are not resolved by this kernel
document.

## Historical implementation and candidate findings

| Work family | Durable result | Status/boundary |
|---|---|---|
| Legacy role/power source audit | GPF1 sampling, inverted aliases, GPF7 pull-up, UDC/OHCI interactions, two-second detector paths, and legacy sysfs defects characterized. | Source correlation; binary identity not established. |
| FSM repair audit | Manual host sysfs path reaches unsafe cleanup and masks errors; transactional ownership and common commit path needed. | Read-only source audit plus documented bench Oops. |
| Authoritative architecture / V001 | Seven-state enum can express one serialized connector-role contract if UDC/OHCI ownership, entry/revocation, and child observation are coordinated. | V001 patch candidate; initially unbuilt. Design proposal, not live behavior. |
| V002 | Exact-baseline startup correction; compiled and linked; reconstructed candidate validated offline. | Later operator report: desktop booted, Gadget did not enumerate. Functional USB failure; state result lacked correlated trace. |
| V003/V004 | Persistent diagnostic logger then exact-shell compatibility correction; USB authority policy unchanged. | Diagnostic candidates, not role fixes. Exact-shell validation found `printf` unavailable as applet/builtin; V004 corrected this. |
| Rake / RUN-2 | Reconciled seven-state contract and preserved raw black-box archive (SHA-256 recorded above). Cumulative dmesg contains no explicit logged admission failure, `last_error` change, or additional committed transition through uptime 311.35 s; 229 sampled polls consistently show IDLE/no `usb0`/no `g_ether` through 316.02 s. | Explicit logged-failure exclusion is bounded to dmesg coverage; 4.67 s poll-only tail, no physical event time/VBUS, and sampling limits remain. A input/producer versus B queue/reconciliation remains open. |
| V005 trace | Correlated raw GPF1, cache/replay, notifier, role input, queue/worker, admission, error and publication points. | Diagnostic; no policy change. Bench trace is bounded by its actual coverage. |
| V005 RUN-3 root cause / V006 | A registered notifier subscriber was erased by a later `buspower_probe` reset; V006 deletes that reset. | Source plus raw trace confirms this V005 RUN-3 cause. It plausibly explains analogous earlier outcomes but does not prove their cause. V006 is built. |
| V007 | Productionized V006 repair with tracing removed; V002 role authority and rootfs retained. | Built/static validated and ready for regression; no V007 deployment/runtime claim. |
| HOST/CLA late-child audit | Retained OHCI can continue enumeration in CLA; only a successfully configured/published direct root child promotes CLA to HOST. No source-proven lost wakeup or authority timer explains a 43 s delay. | Read-only audit of exact V007 candidate source lineage; not general proof of controller health. |

The chronology and cross-run adjudication are preserved in the reconciliation
report outside this repository; the canonical claims above retain only the
surviving findings and evidence limits.

## Negative findings

- `USB_STATE_HOST` does not alone prove VBUS sourcing, a connected child, USB
  traffic, or a working application service.
- `USB_STATE_CLA` does not prove the device is physically absent; it means no
  qualifying child has been published under the relevant contract.
- Legacy `IDLE` did not guarantee both controllers were inactive. A historical
  working Gadget plus `IDLE` observation is possible in Era A; it violates the
  stronger Era B ownership contract rather than defining `IDLE` differently.
- `request_irq()` returning `-EBUSY` establishes contention only; it does not
  identify the IRQ owner or prove Gadget health.
- Unloading UDC/Gadget alone does not source a GPF1 power-off event in the
  audited source. An observed nearby power event could have been sampled or
  queued earlier.
- The V002 offline compile/image pass did not prove runtime Gadget success.
- V003/V004 logger defects and V005/V006 notifier findings are not evidence
  that every earlier missing-Gadget run had the same cause.
- Current live uname/state observations are not matched to any candidate image
  by a kernel hash.

## Known unknowns

- Exact source/config/module correspondence for the current live kernel and
  current installed startup scripts.
- Whether the V007 candidate was ever installed or run; available work-unit
  records say it was not deployed.
- Current live correctness across all role, power, suspend/resume, provider
  failure, and late-child transitions.
- Board-level VBUS sense circuitry and thresholds, and live register values.
- Whether the recorded manual-write Oops faulted specifically at
  `del_timer_sync()`; the source supports the uninitialized timer mechanism,
  but the incident lacks a cryptographically bound kernel and demonstrated
  instruction-level mapping in the evidence reviewed here.
- SMP/preemption and arbitrary hot-unbind behavior beyond the audited UP
  candidate assumptions.
- USB controller/electrical causes when a child is physically attached but
  never reaches successful USB-core configuration/publication.

## Canonical references

- [Tomi device identity](../devices/tt3-tomi.md)
- [TomiDock architecture](../architecture/tomidock.md)
- [Tomi Linux boot chain](../architecture/tomi-boot-chain.md)
- [Tomi bootloader image](tomi-bootloader-image.md)
- [Tomi runtime ABI](tomi-runtime-abi.md)
- [Evidence index](../../evidence/INDEX.md)

## Provenance

This is a synthesis of read-only source audits, candidate/build reports, and
retained bench evidence. The targeted 2026-09-25 coverage reconciliation is
documented outside Git at
`/mnt/d/Codex/TT3/luce-tomidock-usb-lifecycle-coverage-fix-2026-09-25.md`.
That report is a migration work product, not a new canonical LUCE evidence
index. No firmware, device, or source tree was modified for this documentation
pass.
