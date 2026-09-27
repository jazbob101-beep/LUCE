# TomTom Revival — Current State

## Current focus

The primary device is TT3 “Tomi,” with TomiDock providing its current USB/network connection. Active development is **RFNAV**, the RF signal-navigator project built on Tomi + TomiDock. **RFNAV-004 is now physically qualified**: the RFNAV-003 ESP continues to deliver versioned RFN1 target telemetry directly to Tomi over CDC-ECM, and the Tomi-native Nano-X `nxrfnav-v001` application consumes that stream as a live signal instrument display. The UI then remained in continuous operator-observed operation for approximately 12 hours without reported functional instability. The scope of **RFNAV-005** is intentionally not yet fixed; richer target-selection workflow and GPS correlation are the leading next directions. Routine USB detach-anomaly cycling remains retired.

## Stable operational baseline

- **TomiDock is operational.** Tomi is the USB host; the ESP32-S3 is the CDC-ECM device and Wi-Fi routing endpoint. GPIO4 is unconnected and has no runtime VBUS/session role. Normal ECM, routing/NAPT, and TCP/2323 forwarding remain qualified.
- **RFNAV-004 is the qualified RFNAV baseline.** It retains the qualified RFNAV-003 ESP firmware and direct UDP/5515 RFN1 telemetry unchanged, and adds the Tomi-native Nano-X live signal UI.
- **Qualified ESP application:** SHA-256 `30bb4799a0dfb6638a4582eb4c9f01f24f75b9e482c14a8c4147c120884600c4`.
- **Qualified Tomi UI:** `/mnt/sdcard/opentom/rfnav/ui/nxrfnav-v001`, 13,708 bytes, SHA-256 `a7947016ae034199dcbab1240635f503cf2f003226c04e19ca0ccabda90ef365`, built with `arm-linux-gcc (GCC) 3.3.4` against Nano-X.
- **The Tomi diagnostic receiver remains available.** `/mnt/sdcard/opentom/rfnav/bin/rfnav-rx` is 6,512 bytes, SHA-256 `f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6`. It is diagnostic plumbing, not the normal user-facing RFNAV consumer.
- **Routine file movement is documented.** Use the established workflow in the [Tomi File Transfer runbook](docs/runbooks/tomi-file-transfer.md); do not substitute bare SCP as the Tomi-to-MacBook method.
- **Tomi runtime constraints are recorded** in the [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md).
- **Build and machine roles are recorded** in the [Lab Environment reference](docs/environment/lab-environment.md).

## Current support tooling

### RFNAV

RFNAV-002.1 reproduced an ESP-IDF `sys_evt` stack overflow three times with Tomi absent. RFNAV-002.2 increased the configured event-task stack from 2304 to 4096 bytes, 4608 effective in this build. Physical instrumentation measured 1648 bytes worst-case free, implying approximately 2960 bytes maximum observed use, about 144 bytes beyond the old effective allocation. That root cause remains closed.

RFNAV-003 keeps the same qualified event-task allocation and adds a separate priority-1 telemetry sender task behind a bounded 16-entry zero-wait queue. Socket creation, formatting, and `sendto()` do not execute in `sys_evt` context. UDP/5515 targets Tomi at `192.168.77.1`; existing LAN-side UDP/5514 diagnostics remain separate.

Physical RFNAV-003 qualification recorded 484 consecutive Tomi-received RFN1 datagrams, sequence 106 through 589, all from ESP ECM peer `192.168.77.2`, with zero gaps, rewinds, source changes, malformed records, or oversized records. ESP sender diagnostics reached at least 506 successful sends with zero queue drops, send errors, stale drops, or backoff drops while Tomi was present. The same run recorded 538/538 successful target scans and 1,828/1,828 successful LAN pings.

RFNAV-004 adds `nxrfnav-v001`, a direct UDP/5515 Nano-X consumer. Its display provides abbreviated target identity, channel, raw RSSI, a progressive smoothed signal gauge, stronger/steady/weaker trend state, target state, and live/stale/no-telemetry freshness. Gauge mapping is clamped approximately from -90 dBm to -30 dBm; smoothing uses an integer EMA with alpha 0.5; trend uses a short four-reading smoothed history with a +/-2 dB deadband.

The corrected RFNAV-004 source defines `_ISOC99_SOURCE 1` before includes so the pinned glibc 2.3 headers declare `snprintf` correctly. The canonical GCC 3.3.4 rebuild then emitted only the historical Nano-X `index` shadow warning and produced the same target bytes as the earlier build: 13,708 bytes, SHA-256 `a7947016ae034199dcbab1240635f503cf2f003226c04e19ca0ccabda90ef365`.

The live-screen physical smoke test showed current RFN1 data rendering correctly. The UI subsequently remained running for approximately 12 hours with no reported functional instability; the gauge tracked displayed RSSI changes and the weak-to-strong presentation behaved correctly. This endurance result is operator-observed, not a continuous 12-hour diagnostic log, and is recorded with that provenance limit.

See [RF Navigator](docs/architecture/rfnav.md) for protocol, UI behavior, qualification details, evidence identities, and limits.

### PowerFlight

A preserved PowerFlight v001.5.3 run archive has six payloads verified against its embedded manifest and no `BLOCKED` events; its Tomi-specific records strongly support a physical Tomi run, but the acquisition path and exact recorder bytes are not bound. Its final recorded state was `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, `BATTERY_RETURN_CHILD_ATTACHED`. See the [power/battery reference](docs/reference/tomi-power-battery.md) for provenance limits.

### nxbattery

Current operator-reported status is that nxbattery V002 is deployed at `/mnt/sdcard/opentom/ui/nxbattery-v002`. The preserved V002 build is 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. With external power detected, suppression of percentage/bar fill is intentional; voltage and raw `CHARGE=<integer>` remain visible.

### Diagnostics

The Phase 4F reduced observer remains diagnostic-only, not production architecture. Preserve it for opportunistic capture only if an unrelated anomaly occurs.

## Closed decisions

- RFNAV-004 is physically qualified and supersedes RFNAV-003 as the active RFNAV baseline.
- RFNAV-003 ESP firmware and RFN1 protocol remain the qualified transport beneath RFNAV-004; 004 did not require an ESP or protocol revision.
- `nxrfnav-v001` is the qualified normal user-facing RFNAV consumer; `rfnav-rx` remains diagnostic-only.
- RFNAV-002's unsolicited reset root cause is closed as `sys_evt` stack exhaustion under the targeted-scan workload. Keep the configured event-task stack at 4096 bytes unless a later measured workload justifies change.
- RFNAV Tomi-facing telemetry is direct UDP over ECM, ESP `192.168.77.2` to Tomi `192.168.77.1:5515`; do not reroute the RFN1 product path through the Wi-Fi LAN.
- Keep telemetry socket/formatting work out of `sys_evt` / Wi-Fi callback context.
- RSSI is a received-signal-strength indicator, not a distance measurement.
- GPIO4 remains removed/unconnected and has no RFNAV role.
- Role-aware Tomi boot integration already exists. Do not revive a proposed boot-integration v002.
- Never issue `echo host > /sys/devices/platform/tomtomgo-usbmode/mode`; prior use caused rc=139/kernel Oops behavior.
- Do not connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. See [TomiDock Architecture](docs/architecture/tomidock.md) for the complete safe sequence.

## Open / next work

- **RFNAV-005:** scope is not yet adjudicated. The two leading product directions are richer operator target-selection workflow and GPS correlation/navigation context. Choose the next milestone explicitly rather than combining both by default.
- GPS-based directional navigation, mapping, persistence, and BLE remain future work unless separately promoted into scope.
- Deterministic no-GPIO physical-detach fail-close remains formally unproven; historical A8/premature-MOUNT observations remain closed as an unrelated unresolved lifecycle boundary.
- Exact installed provenance for the current `tomidock-netd` and `/etc/rc` remains unbound. The qualified RFNAV ESP and Tomi UI binaries have recorded build identities, but no independent post-install readback of every deployed component is asserted here.
- The exact Tomi VBUS-switch implementation and formal USB self-powered compliance remain undocumented/unestablished.

## Canonical map

- [TT3 “Tomi” device profile](docs/devices/tt3-tomi.md) — device identity and stable hardware facts.
- [TomiDock Architecture](docs/architecture/tomidock.md) — physical/USB/network design, lifecycle, and safety constraints.
- [RF Navigator](docs/architecture/rfnav.md) — RFNAV behavior, protocol, UI, qualification state, root-cause closure, and staged development.
- [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md) — Tomi commands and compatibility.
- [Tomi File Transfer](docs/runbooks/tomi-file-transfer.md) — current transfer procedures.
- [Lab Environment](docs/environment/lab-environment.md) — machine roles, source/build paths, and command contexts.
- [Evidence Index](evidence/INDEX.md) — conventions for locating preserved external evidence.

Detailed reports and raw experiment artifacts remain in `/mnt/d/Codex`; they are evidence stores, not substitutes for current LUCE decisions.
