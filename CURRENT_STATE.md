# TomTom Revival — Current State

## Current focus

The primary device is TT3 “Tomi,” with TomiDock providing its current USB/network connection. Active development is **RFNAV**, the RF signal-navigator project built on Tomi + TomiDock. **RFNAV-003 is now physically qualified**: the ESP delivers versioned RFN1 target telemetry directly to Tomi over the existing CDC-ECM link, and a Tomi-native receiver built with the pinned GCC 3.3.4 toolchain captured a lossless sequence window from the ESP ECM peer. The next planned milestone is **RFNAV-004**, a small user-facing Tomi signal display consuming the qualified RFN1 stream. Routine USB detach-anomaly cycling remains retired.

## Stable operational baseline

- **TomiDock is operational.** Tomi is the USB host; the ESP32-S3 is the CDC-ECM device and Wi-Fi routing endpoint. GPIO4 is unconnected and has no runtime VBUS/session role. Normal ECM, routing/NAPT, and TCP/2323 forwarding remain qualified.
- **RFNAV-003 is the qualified RFNAV baseline.** It retains RFNAV-002.2 discovery/target tracking and the qualified `sys_evt` stack repair, and adds direct ESP-to-Tomi UDP/5515 RFN1 telemetry over the `.77.x` ECM subnet. Qualified ESP application SHA-256: `30bb4799a0dfb6638a4582eb4c9f01f24f75b9e482c14a8c4147c120884600c4`.
- **The Tomi receiver is qualified.** `/mnt/sdcard/opentom/rfnav/bin/rfnav-rx` is 6,512 bytes, SHA-256 `f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6`, built with `arm-linux-gcc (GCC) 3.3.4`. It is diagnostic plumbing for the RFN1 transport, not the final UI.
- **Routine file movement is documented.** Use the established workflow in the [Tomi File Transfer runbook](docs/runbooks/tomi-file-transfer.md); do not substitute bare SCP as the Tomi-to-MacBook method.
- **Tomi runtime constraints are recorded** in the [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md).
- **Build and machine roles are recorded** in the [Lab Environment reference](docs/environment/lab-environment.md).

## Current support tooling

### RFNAV

RFNAV-002.1 reproduced an ESP-IDF `sys_evt` stack overflow three times with Tomi absent. RFNAV-002.2 increased the configured event-task stack from 2304 to 4096 bytes, 4608 effective in this build. Physical instrumentation measured 1648 bytes worst-case free, implying approximately 2960 bytes maximum observed use, about 144 bytes beyond the old effective allocation. That root cause remains closed.

RFNAV-003 keeps the same qualified event-task allocation and adds a separate priority-1 telemetry sender task behind a bounded 16-entry zero-wait queue. Socket creation, formatting, and `sendto()` do not execute in `sys_evt` context. UDP/5515 targets Tomi at `192.168.77.1`; existing LAN-side UDP/5514 diagnostics remain separate.

Physical RFNAV-003 qualification recorded 484 consecutive Tomi-received RFN1 datagrams, sequence 106 through 589, all from ESP ECM peer `192.168.77.2`, with zero gaps, rewinds, source changes, malformed records, or oversized records. ESP sender diagnostics reached at least 506 successful sends with zero queue drops, send errors, stale drops, or backoff drops while Tomi was present. The sender stack retained 2104 bytes free, `sys_evt` retained 1648 bytes free, and the RFNAV worker retained 1148 bytes free. The same run recorded 538/538 successful target scans and 1,828/1,828 successful LAN pings.

See [RF Navigator](docs/architecture/rfnav.md) for wire format, qualification metrics, evidence identities, and limits.

### PowerFlight

A preserved PowerFlight v001.5.3 run archive has six payloads verified against its embedded manifest and no `BLOCKED` events; its Tomi-specific records strongly support a physical Tomi run, but the acquisition path and exact recorder bytes are not bound. Its final recorded state was `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, `BATTERY_RETURN_CHILD_ATTACHED`. See the [power/battery reference](docs/reference/tomi-power-battery.md) for provenance limits.

### nxbattery

Current operator-reported status is that nxbattery V002 is deployed and running at `/mnt/sdcard/opentom/ui/nxbattery-v002`. The preserved V002 build is 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. With external power detected, suppression of percentage/bar fill is intentional; voltage and raw `CHARGE=<integer>` remain visible.

### Diagnostics

The Phase 4F reduced observer remains diagnostic-only, not production architecture. Preserve it for opportunistic capture only if an unrelated anomaly occurs.

## Closed decisions

- RFNAV-003 is physically qualified and supersedes RFNAV-002.2 as the active RFNAV baseline.
- RFNAV-002's unsolicited reset root cause is closed as `sys_evt` stack exhaustion under the targeted-scan workload. Keep the configured event-task stack at 4096 bytes unless a later measured workload justifies change.
- RFNAV Tomi-facing telemetry is direct UDP over ECM, ESP `192.168.77.2` to Tomi `192.168.77.1:5515`; do not reroute the RFN1 product path through the Wi-Fi LAN.
- Keep telemetry socket/formatting work out of `sys_evt` / Wi-Fi callback context.
- GPIO4 remains removed/unconnected and has no RFNAV role.
- Role-aware Tomi boot integration already exists. Do not revive a proposed boot-integration v002.
- Never issue `echo host > /sys/devices/platform/tomtomgo-usbmode/mode`; prior use caused rc=139/kernel Oops behavior.
- Do not connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. See [TomiDock Architecture](docs/architecture/tomidock.md) for the complete safe sequence.

## Open / next work

- **RFNAV-004:** build the first real Tomi-facing RFNAV display on top of the qualified RFN1 stream. Keep the first milestone small: current target identity/state, channel, RSSI, and an obvious live signal-strength/trend presentation. GPS correlation, directional navigation, mapping, persistence, BLE, and richer target-selection UX remain later stages unless separately promoted into scope.
- Deterministic no-GPIO physical-detach fail-close remains formally unproven; historical A8/premature-MOUNT observations remain closed as an unrelated unresolved lifecycle boundary.
- Exact installed provenance for the current `tomidock-netd` and `/etc/rc` remains unbound. RFNAV-003 has a qualified application build hash and operator app-only deployment history, but no independent post-flash readback was preserved.
- The exact Tomi VBUS-switch implementation and formal USB self-powered compliance remain undocumented/unestablished.

## Canonical map

- [TT3 “Tomi” device profile](docs/devices/tt3-tomi.md) — device identity and stable hardware facts.
- [TomiDock Architecture](docs/architecture/tomidock.md) — physical/USB/network design, lifecycle, and safety constraints.
- [RF Navigator](docs/architecture/rfnav.md) — RFNAV behavior, protocol, qualification state, root-cause closure, and staged development.
- [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md) — Tomi commands and compatibility.
- [Tomi File Transfer](docs/runbooks/tomi-file-transfer.md) — current transfer procedures.
- [Lab Environment](docs/environment/lab-environment.md) — machine roles, source/build paths, and command contexts.
- [Evidence Index](evidence/INDEX.md) — conventions for locating preserved external evidence.

Detailed reports and raw experiment artifacts remain in `/mnt/d/Codex`; they are evidence stores, not substitutes for current LUCE decisions.
