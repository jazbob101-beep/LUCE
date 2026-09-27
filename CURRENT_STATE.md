# TomTom Revival — Current State

## Current focus

The primary device is TT3 “Tomi,” with TomiDock providing its current USB/network connection. Active development has moved to **RFNAV**, the RF signal-navigator project built on Tomi + TomiDock. RFNAV-002.2 is physically qualified and closes the targeted-scan stability problem; the next planned milestone is **RFNAV-003**, a Tomi-facing telemetry path over the existing ECM link. Routine USB detach-anomaly cycling remains retired. Preserve a newly occurring anomaly if it appears during otherwise authorized work, but do not treat testing as an objective by itself.

## Stable operational baseline

- **TomiDock is operational.** Tomi is the USB host; the ESP32-S3 is the CDC-ECM device and Wi-Fi routing endpoint. The GPIO4 HIGH jumper has been removed; GPIO4 is unconnected and has no runtime VBUS/session role. Normal boot, enumeration, ECM, routing/NAPT, and TCP/2323 forwarding passed the post-removal smoke test. The live Tomi boot path is role-aware and already starts `tomidock-netd`; a new boot-integration project is not needed.
- **RFNAV-002.2 is the qualified RF scanning baseline.** It performs full Wi-Fi discovery, deterministic target selection, and repeated focused target RSSI scans while preserving normal TomiDock networking. Its qualified application SHA-256 is `35dbfa84259b2ca37e534154b43f86401df00dd8c2b759b59b487658ea818672`. Standalone and full TomiDock testing passed after increasing the ESP-IDF system-event task stack. See [RF Navigator](docs/architecture/rfnav.md) for the root-cause arithmetic, qualification metrics, and provenance.
- **Routine file movement is documented.** Use the established workflow in the [Tomi File Transfer runbook](docs/runbooks/tomi-file-transfer.md); do not substitute bare SCP as the Tomi-to-MacBook method.
- **Tomi runtime constraints are recorded** in the [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md). Do not reproduce its BusyBox inventories here.
- **Build and machine roles are recorded** in the [Lab Environment reference](docs/environment/lab-environment.md).

## Current support tooling

### RFNAV

RFNAV-002.1 reproduced an ESP-IDF `sys_evt` stack overflow three times with Tomi physically absent, at approximately 332.5, 204.8, and 192.1 seconds of uptime. The old effective event-task stack was 2816 bytes. RFNAV-002.2 increases the configured event-task stack to 4096 bytes, 4608 effective in this build, and measures a worst-case high-water mark of 1648 free bytes, implying about 2960 bytes maximum observed use. That is about 144 bytes beyond the old allocation and closes the reset root cause as event-task stack exhaustion under the repeated target-scan workload.

RFNAV-002.2 then passed standalone qualification and an approximately 24-minute full TomiDock run with 674/674 target scans completing successfully, zero RFNAV rediscoveries, 2,269/2,269 ping samples successful, ECM/routing/NAPT/TCP/2323 continuously ready, and stable RFNAV/event-task stack and heap telemetry. RFNAV-002 is closed; use RFNAV-002.2 as the parent for RFNAV-003.

### PowerFlight

A preserved PowerFlight v001.5.3 run archive has six payloads verified against its embedded manifest and no `BLOCKED` events; its Tomi-specific records strongly support a physical Tomi run, but the acquisition path and exact recorder bytes are not bound. Its final recorded state was `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, `BATTERY_RETURN_CHILD_ATTACHED`. This is an archive observation, not a universal charging or electrical guarantee. See the [power/battery reference](docs/reference/tomi-power-battery.md) for provenance limits.

### nxbattery

Current operator-reported status is that nxbattery V002 is deployed and running at `/mnt/sdcard/opentom/ui/nxbattery-v002`. The preserved V002 build is 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. With external power detected, suppression of percentage/bar fill (the blank gauge) is intentional; voltage and the raw `CHARGE=<integer>` line remain visible. No formal Run #5 is required for normal current use; further battery-curve work is not a deployment gate.

### Diagnostics

The Phase 4F reduced observer is diagnostic-only, not the production architecture. Preserve it for opportunistic capture if installed and an unrelated anomaly occurs; its current installation state is not asserted here.

## Closed decisions

- RFNAV-002 is complete. RFNAV-002.2 is the working baseline; do not create a cleanup-only revision before RFNAV-003.
- The RFNAV-002.1 unsolicited reset root cause is closed as `sys_evt` stack exhaustion under the targeted-scan workload. The qualified 4096-byte configured event-task stack leaves 1648 bytes of measured worst-case free margin in the current build.
- GPIO4 is not required for the current design; it remains removed/unconnected. Routine GPIO4 detach-cycle testing is retired.
- Role-aware Tomi boot integration already exists. Do not revive a proposed boot-integration v002.
- Never issue `echo host > /sys/devices/platform/tomtomgo-usbmode/mode`; prior use caused rc=139/kernel Oops behavior.
- Do not connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. See [TomiDock Architecture](docs/architecture/tomidock.md) for the complete safe sequence.

## Open / unresolved

- RFNAV-003 has not yet been implemented. Its first milestone is a stable Tomi-facing telemetry path over the existing ECM link carrying target state/channel/RSSI plus sequence or generation information; GPS, Nano-X UI, BLE, mapping, and navigation logic remain later work.
- Deterministic no-GPIO physical-detach fail-close is not formally proven. The A8 missed-fail-close and two premature-MOUNT observations remain unresolved historical findings; they do not reopen the GPIO4 decision.
- The nxbattery build package predates deployment and explicitly recorded “not deployed.” Its current deployment/running status above is operator-reported; no later installation record tying the on-device bytes to the preserved build hash was located. Exact installed provenance for the current `tomidock-netd` and `/etc/rc` remains unbound. RFNAV-002.2 has a qualified application build hash and operator app-only deployment history, but no independent post-flash readback was preserved.
- The exact Tomi VBUS-switch implementation and formal USB self-powered compliance remain undocumented/unestablished.

## Canonical map

- [TT3 “Tomi” device profile](docs/devices/tt3-tomi.md) — device identity and stable hardware facts.
- [TomiDock Architecture](docs/architecture/tomidock.md) — physical/USB/network design, lifecycle, and safety constraints.
- [RF Navigator](docs/architecture/rfnav.md) — RFNAV behavior, qualification state, root-cause closure, and staged development.
- [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md) — Tomi commands and compatibility.
- [Tomi File Transfer](docs/runbooks/tomi-file-transfer.md) — current transfer procedures.
- [Lab Environment](docs/environment/lab-environment.md) — machine roles, source/build paths, and command contexts.
- [Evidence Index](evidence/INDEX.md) — conventions for locating preserved external evidence.

Detailed reports and raw experiment artifacts remain in `/mnt/d/Codex`; they are evidence stores, not substitutes for current LUCE decisions.
