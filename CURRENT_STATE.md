# TomTom Revival — Current State

## Current focus

The primary device is TT3 “Tomi,” with TomiDock providing its current USB/network connection. No new bench experiment is presently designated. Use the operational baseline below and the canonical references for details; routine USB detach-anomaly cycling is retired. Preserve a newly occurring anomaly if it appears during otherwise authorized work, but do not treat testing as an objective by itself.

## Stable operational baseline

- **TomiDock is operational.** Tomi is the USB host; the ESP32-S3 is the CDC-ECM device and Wi-Fi routing endpoint. The GPIO4 HIGH jumper has been removed; GPIO4 is unconnected and has no runtime VBUS/session role. Normal boot, enumeration, ECM, routing/NAPT, and TCP/2323 forwarding passed the post-removal smoke test. The live Tomi boot path is role-aware and already starts `tomidock-netd`; a new boot-integration project is not needed.
- **Routine file movement is documented.** Use the established workflow in the [Tomi File Transfer runbook](docs/runbooks/tomi-file-transfer.md); do not substitute bare SCP as the Tomi-to-MacBook method.
- **Tomi runtime constraints are recorded** in the [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md). Do not reproduce its BusyBox inventories here.
- **Build and machine roles are recorded** in the [Lab Environment reference](docs/environment/lab-environment.md).

## Current support tooling

### PowerFlight

A preserved PowerFlight v001.5.3 run archive has six payloads verified against its embedded manifest and no `BLOCKED` events; its Tomi-specific records strongly support a physical Tomi run, but the acquisition path and exact recorder bytes are not bound. Its final recorded state was `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, `BATTERY_RETURN_CHILD_ATTACHED`. This is an archive observation, not a universal charging or electrical guarantee. See the [power/battery reference](docs/reference/tomi-power-battery.md) for provenance limits.

### nxbattery

Current operator-reported status is that nxbattery V002 is deployed and running at `/mnt/sdcard/opentom/ui/nxbattery-v002`. The preserved V002 build is 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. With external power detected, suppression of percentage/bar fill (the blank gauge) is intentional; voltage and the raw `CHARGE=<integer>` line remain visible. No formal Run #5 is required for normal current use; further battery-curve work is not a deployment gate.

### Diagnostics

The Phase 4F reduced observer is diagnostic-only, not the production architecture. Preserve it for opportunistic capture if installed and an unrelated anomaly occurs; its current installation state is not asserted here.

## Closed decisions

- GPIO4 is not required for the current design; it remains removed/unconnected. Routine GPIO4 detach-cycle testing is retired.
- Role-aware Tomi boot integration already exists. Do not revive a proposed boot-integration v002.
- Never issue `echo host > /sys/devices/platform/tomtomgo-usbmode/mode`; prior use caused rc=139/kernel Oops behavior.
- Do not connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. See [TomiDock Architecture](docs/architecture/tomidock.md) for the complete safe sequence.

## Open / unresolved

- Deterministic no-GPIO physical-detach fail-close is not formally proven. The A8 missed-fail-close and two premature-MOUNT observations remain unresolved historical findings; they do not reopen the GPIO4 decision.
- The nxbattery build package predates deployment and explicitly recorded “not deployed.” Its current deployment/running status above is operator-reported; no later installation record tying the on-device bytes to the preserved build hash was located. Exact installed provenance for the current `tomidock-netd`, `/etc/rc`, and ESP image is also not hash-bound in the available records.
- The exact Tomi VBUS-switch implementation and formal USB self-powered compliance remain undocumented/unestablished.

## Canonical map

- [TT3 “Tomi” device profile](docs/devices/tt3-tomi.md) — device identity and stable hardware facts.
- [TomiDock Architecture](docs/architecture/tomidock.md) — physical/USB/network design, lifecycle, and safety constraints.
- [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md) — Tomi commands and compatibility.
- [Tomi File Transfer](docs/runbooks/tomi-file-transfer.md) — current transfer procedures.
- [Lab Environment](docs/environment/lab-environment.md) — machine roles, source/build paths, and command contexts.
- [Evidence Index](evidence/INDEX.md) — conventions for locating preserved external evidence.

Detailed reports and raw experiment artifacts remain in `/mnt/d/Codex`; they are evidence stores, not substitutes for current LUCE decisions.
