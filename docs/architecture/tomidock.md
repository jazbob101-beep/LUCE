# TomiDock Architecture

## Purpose

TomiDock connects TT3 “Tomi” to a local Wi-Fi LAN through an independently powered ESP32-S3. The ESP presents a USB CDC-ECM device to Tomi, routes between the USB network and Wi-Fi, and exposes Tomi's shell through one explicit TCP forward. This page records the durable topology, responsibilities, decisions, and constraints; device identity, Tomi command compatibility, and file-transfer procedures remain in their own canonical documents.

## System overview

```text
LAN clients
    |
Wi-Fi access point / LAN
    |
ESP32-S3 / TomiDock
  Wi-Fi: 192.168.1.223
  routing / NAPT
  TCP 2323 -> 192.168.77.1:23
  CDC-ECM: 192.168.77.2
    |
  USB D+ / D- / GND; Tomi is host
  Tomi VBUS branch is separately switchable
    |
Tomi
  USB host; usb0: 192.168.77.1
```

The network roles and addresses are distinct: `192.168.1.223` is the ESP's LAN-side address, `192.168.77.2` is its ECM-side address, and `192.168.77.1` is Tomi's ECM-side address. The normal LAN shell path is `192.168.1.223:2323` forwarded to Tomi TCP/23. This is routed/NAPT connectivity with an explicit inbound service forward, not a claim of transparent Wi-Fi-to-USB Ethernet bridging or unrestricted inbound TCP access.

## Physical topology and power

The ESP32-S3 is independently supplied from bench 5 V. Tomi's VBUS branch is separately switchable while the USB data connection remains present when the Tomi USB cable is connected. The preserved records establish this separate VBUS control at the rig level but do not establish the switch component, electrical implementation, or its controller; do not infer those details.

GPIO4 has no role in the current design:

```text
GPIO4 HIGH jumper:                removed
GPIO4 electrical state:           unconnected
GPIO4 runtime VBUS/session role:  none
External GPIO4/BVALID monitor:    not used
```

Normal ESP boot, enumeration as `303a:4002`, CDC-ECM, Wi-Fi/routing/NAPT, and TCP/2323 forwarding passed the post-removal smoke test. This closes the question of whether the HIGH jumper is required for normal intended operation; it does not prove deterministic physical-detach fail-close behavior or formal USB electrical compliance.

## USB architecture and lifecycle

Tomi operates as the USB host. The ESP32-S3 operates as the native USB CDC-ECM device and has been observed by Tomi as VID:PID `303a:4002`, product string `TomiDock CDC ECM v001`. The intended ESP startup explicitly connects the device after successful TinyUSB driver installation; the no-external-monitor configuration does not consume GPIO4 as BVALID/VBUS input.

The application gates routing eligibility on the USB lifecycle. In the production behavior under review, suspend makes ECM routing unready, which disables forwarding/NAPT and TCP/2323; a valid mount makes routing eligible again. A mount is an application lifecycle event, not by itself proof of a physically present host beyond the USB stack's event semantics.

Historical observations include one missed application fail-close during a marked physical absence and two apparent premature MOUNT events during reported cable absence. Later instrumented campaigns did not reproduce those phenotypes consistently. They remain unresolved lifecycle robustness observations: deterministic no-GPIO physical-detach fail-close is not proven. They do not establish that GPIO4 is required, and routine detach-cycle hunting is retired. Preserve a future unsolicited anomaly if one occurs, but do not promote these historical observations into current GPIO4 requirements.

## Network architecture

The ESP's Wi-Fi interface is the LAN endpoint (`192.168.1.223`); its CDC-ECM interface peers with Tomi on `192.168.77.0/24`. Routing/NAPT provides the tested LAN-to-TomiDock connectivity. A specific inbound mapping exposes ESP LAN TCP/2323 to Tomi `192.168.77.1:23`. This single mapping is the documented inbound service; do not infer that arbitrary Tomi TCP services are transparently reachable from the LAN.

The `.77.2` address is the ESP ECM peer, useful for Tomi-to-ESP traffic; it is not a MacBook/LAN service address. Tomi's stable identity and role-specific details are in [the TT3 device profile](../devices/tt3-tomi.md). Routine network file movement belongs in [the Tomi file-transfer runbook](../runbooks/tomi-file-transfer.md), and command/runtime constraints belong in [the Tomi Runtime ABI](../reference/tomi-runtime-abi.md).

## Tomi-side service integration

The supplied 2026-09-24 live-state capture reports `/mnt/sdcard/opentom/start.sh` starting `/mnt/sdcard/opentom/tomidock/bin/tomidock-netd`, independently of the desktop lifecycle. It reports current `/etc/rc` USB handling as role-aware, with no forced-host command required. These are live operator observations; the exact installed scripts/binary and their hashes were not preserved in the inspected local evidence, so this page does not claim source-to-installed-image identity.

Never use this Tomi command as a shortcut:

```sh
echo host > /sys/devices/platform/tomtomgo-usbmode/mode
```

Prior use caused rc=139 / kernel Oops behavior. Retain the existing role-aware boot/service path; do not revive a forced-host startup proposal.

## PowerFlight

PowerFlight is a Tomi-side lifecycle/power observation and qualification tool, not a TomiDock network service and not part of the ESP forwarding path. Its rebuilt-rig model distinguishes battery return from cable removal: Tomi VBUS can be off while the USB child remains present under `USB_STATE_HOST` because data remains connected.

The v001.5.3 formal run recorded `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, and `BATTERY_RETURN_CHILD_ATTACHED`; the run manifest verifies all six captured files. The archive is `/mnt/d/Codex/TT3/Evidence/powerflight-20260921-170744-7150-00.tar.gz`, SHA-256 `67a8c41caa80c09ebe2fe9114bd01ba00b5ab84343b6ef750ccb58c5e52daf8a`. This is evidence for the observed lifecycle state, not a universal electrical charging guarantee or proof of the VBUS switch implementation.

## Durable design decisions

- GPIO4 is removed from current hardware requirements, left unconnected, and not consumed by firmware as VBUS/session validity.
- The ESP is independently powered; Tomi VBUS is separately switchable at the bench topology level.
- Tomi is the USB host; the ESP is the CDC-ECM device and Wi-Fi routing endpoint.
- Routing readiness follows USB lifecycle state and fails closed on observed suspend; historical intermittent lifecycle findings remain a separate limitation.
- LAN access to Tomi's shell is the explicit TCP/2323-to-TCP/23 mapping, not a general inbound forwarding promise.
- Tomi boot integration is role-aware; do not add a new boot integration merely because older proposals predate the live setup.

## Safety constraints

### ESP debug/programming USB and backpower

Do not connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock USB path. A backpower path was bench-proven. Use the isolated sequence: disconnect Tomi; turn external ESP bench 5 V off; connect programming USB and verify the serial device; flash; disconnect programming USB; restore bench power; then reconnect Tomi. Opening the debug serial interface may reset the ESP. The detailed file-transfer and Tomi runtime procedures remain in their canonical references rather than being duplicated here.

### Evidence and implementation identity

Do not equate a source tree, disposable build, or diagnostic image with the currently installed ESP image without a matching identity record. The Phase 4F reduced observer is diagnostic-only; it may be useful for opportunistic anomaly capture if installed, but it is not the production architecture. The post-jumper-removal smoke result is not tied to an independently recorded firmware hash in the inspected evidence.

## Known limitations and unresolved observations

- The exact circuit/component/controller that switches Tomi VBUS is not established by the inspected records.
- Exact installed hashes/source provenance for the current `tomidock-netd`, role-aware `/etc/rc`, and ESP firmware used during the post-removal smoke are not established.
- A8 missed fail-close and two premature-MOUNT reports remain accepted but unresolved; deterministic no-GPIO detach behavior is unproven. Routine anomaly cycling is retired unless a new event or requirement justifies reopening it.
- Formal USB self-powered compliance and broader electrical/backpower qualification are not established by the functional smoke test.
- The evidence establishes the explicit TCP/2323 inbound mapping; broader configurable or transparent LAN-to-ECM TCP ingress policy is not established.

## Canonical references

- [TT3 “Tomi” device profile](../devices/tt3-tomi.md)
- [Tomi Runtime ABI](../reference/tomi-runtime-abi.md)
- [Tomi file-transfer runbook](../runbooks/tomi-file-transfer.md)
- [Current project state](../../CURRENT_STATE.md)

## Provenance

The current physical/GPIO4 decision and post-removal smoke results are documented in `/mnt/d/Codex/TT3/20_GPIO4_JUMPER_REMOVAL_AND_DESIGN_CLOSEOUT_2026-09-24.md`. Network topology, Tomi-side live service/role-aware boot observations, and programming safety are in `/mnt/d/Codex/TomiDock_Fresh_Thread_Handoff_2026-09-24.md` and the operator-provided 2026-09-24 live-state capture. USB lifecycle findings are summarized in the Phase 4F/4G reports under `/mnt/d/Codex/TT3/tomidock-gpio4-bvalid-audit-20260921/`. The PowerFlight claim points to the hash-identified formal run archive above and `/mnt/d/Codex/TT3/tomi-powerflight-v001.5.3-20260921/REPORT.md`. Detailed source paths and evidence boundaries are recorded in the external reconciliation report for this migration.
