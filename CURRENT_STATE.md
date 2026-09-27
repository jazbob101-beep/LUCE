# TomTom Revival — Current State

## Current focus

The primary device is TT3 “Tomi,” with TomiDock providing its current USB/network connection. Active development is **RFNAV**, the RF signal-navigator project built on Tomi + TomiDock. **RFNAV-005 is now physically qualified**: the qualified ESP firmware retains direct RFN1 tracking telemetry over CDC-ECM and adds a recoverable RFC1/RFD1/RFA1 discovery/control plane; the Tomi-native Nano-X `nxrfnav-v002` application integrates with the desktop, discovers nearby APs, provides paged touchscreen selection, tracks a chosen target live, supports target changes, closes cleanly, and relaunches correctly. A bounded real ECM loss/recovery cycle also passed. Routine USB detach-anomaly cycling remains retired.

## Stable operational baseline

- **TomiDock is operational.** Tomi is the USB host; the ESP32-S3 is the CDC-ECM device and Wi-Fi routing endpoint. GPIO4 is unconnected and has no runtime VBUS/session role. Normal ECM, routing/NAPT, and TCP/2323 forwarding remain qualified.
- **RFNAV-005 is the qualified RFNAV baseline.** It retains RFN1 target telemetry on UDP/5515 and adds Tomi-to-ESP RFC1 commands on UDP/5516 plus ESP-to-Tomi RFD1/RFA1 replies on UDP/5517.
- **Qualified ESP application:** 818,848 bytes, SHA-256 `1d8d1bce5e30f0f3ccc5ef8f0e03df1372bbe69729f1b46d8d35cfcadec83a09`.
- **Qualified Tomi UI:** `/mnt/sdcard/opentom/rfnav/ui/nxrfnav-v002`, 20,148 bytes, SHA-256 `865cf76ddbe11ea1a383eea0ef107f23fbe85399d92933257317f28fbc680ba2`, built with `arm-linux-gcc (GCC) 3.3.4` against Nano-X.
- **Qualified merged launcher:** `/mnt/sdcard/opentom/thomas/etc/nxlaunch.cnf`, 217 bytes, SHA-256 `54651633d90457e55425ac43ffe9b8ac6f892d2e79fad823937027ed972d6057`; the RFNAV entry launches the qualified UI without replacing the existing GPS entry.
- **The Tomi diagnostic receiver remains available.** `/mnt/sdcard/opentom/rfnav/bin/rfnav-rx` is 6,512 bytes, SHA-256 `f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6`. It is diagnostic plumbing, not the normal user-facing RFNAV consumer.
- **Routine file movement is documented.** Use the established workflow in the [Tomi File Transfer runbook](docs/runbooks/tomi-file-transfer.md); do not substitute bare SCP as the Tomi-to-MacBook method.
- **Tomi runtime constraints are recorded** in the [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md), including the canonical OpenTom reboot helper rather than the inert BusyBox reboot applet.
- **Build and machine roles are recorded** in the [Lab Environment reference](docs/environment/lab-environment.md).

## Current support tooling

### RFNAV

RFNAV-002.1 reproduced an ESP-IDF `sys_evt` stack overflow three times with Tomi absent. RFNAV-002.2 increased the configured event-task stack from 2304 to 4096 bytes, 4608 effective in this build. Physical instrumentation measured 1648 bytes worst-case free, implying approximately 2960 bytes maximum observed use, about 144 bytes beyond the old effective allocation. That root cause remains closed, and RFNAV-005 retains the 4096-byte configured event-task stack.

RFNAV-003 established the separate priority-1 RFN1 telemetry sender behind a bounded 16-entry zero-wait queue. Socket creation, formatting, and `sendto()` do not execute in `sys_evt` context. UDP/5515 targets Tomi at `192.168.77.1`; LAN-side UDP/5514 diagnostics remain separate. Physical RFNAV-003 qualification recorded 484 consecutive Tomi-received RFN1 datagrams with zero gaps, rewinds, source changes, malformed records, or oversized records, plus 538/538 successful target scans and 1,828/1,828 successful LAN pings.

RFNAV-004 added the Nano-X live signal model and physically qualified target identity, channel, raw RSSI, smoothed gauge, trend, and freshness behavior, including an approximately 12-hour operator-observed endurance run.

RFNAV-005 adds the product workflow around that qualified signal model. DISCOVER returns a bounded strongest-first AP list; PREV/NEXT provide paged touchscreen navigation; RESCAN refreshes the discovery set; SELECT is authorized against recent same-epoch discovery state; TRACK uses the existing RFN1 stream; Change Target returns to discovery without competing target ownership; CLOSE releases resources; duplicate application instances are rejected; and UDP/5515 is released for the diagnostic receiver after exit.

The RFNAV-005 control service has one socket owner, bounded reply handling, lifecycle recovery, and low-frequency `RFNAV_CTRL: HEALTH` counters. The original pre-repair candidate timed out during the first physical DISCOVER attempt. Independent review rejected the proposed early-bind/address-absence explanation as stated and found several real defects: transient setup failure could permanently kill the control task, control initialization failure could bypass focused-scan delay, replay could reference stale/overwritten discovery state, Tomi could accept overflowing request IDs, and partial discovery lists could remain selectable after failure. Those defects were repaired and covered by new regression tests.

The exact cause of the original pre-repair physical timeout remains unresolved because that run lacked the listener and request/reply discriminator data needed to bind it to one repaired defect. Do not treat the later successful repair as proof of a specific historical cause.

RFNAV-005 physical qualification deliberately included ESP startup before Tomi/ECM availability, real multi-page DISCOVER, RESCAN, PREV/NEXT, two non-associated target selections, live TRACK behavior, Change Target, CLOSE, relaunch and a second full examination, duplicate-instance rejection, UDP/5515 cleanup, and one bounded ECM loss/recovery cycle. Instrumented capture showed control `ready=1` before the cycle, `waiting_for_ecm` and `ready=0` during the real ECM-unready interval, then `ready local=192.168.77.2:5516`, a new ECM generation, restored routing/NAPT/TCP2323, and successful post-recovery manual discovery. Associated-uplink AP selection was intentionally not exercised and is outside the RFNAV-005 qualification claim.

See [RF Navigator](docs/architecture/rfnav.md) for protocol, UI behavior, review findings, qualification details, evidence identities, and limits.

### PowerFlight

A preserved PowerFlight v001.5.3 run archive has six payloads verified against its embedded manifest and no `BLOCKED` events; its Tomi-specific records strongly support a physical Tomi run, but the acquisition path and exact recorder bytes are not bound. Its final recorded state was `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, `BATTERY_RETURN_CHILD_ATTACHED`. See the [power/battery reference](docs/reference/tomi-power-battery.md) for provenance limits.

### nxbattery

Current operator-reported status is that nxbattery V002 is deployed at `/mnt/sdcard/opentom/ui/nxbattery-v002`. The preserved V002 build is 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. With external power detected, suppression of percentage/bar fill is intentional; voltage and raw `CHARGE=<integer>` remain visible.

### Diagnostics

The Phase 4F reduced observer remains diagnostic-only, not production architecture. Preserve it for opportunistic capture only if an unrelated anomaly occurs.

## Closed decisions

- **RFNAV-005 is physically qualified and supersedes RFNAV-004 as the active RFNAV baseline.**
- `nxrfnav-v002` is the qualified normal user-facing RFNAV consumer; `rfnav-rx` remains diagnostic-only.
- RFNAV-003 RFN1 telemetry remains the tracking transport beneath RFNAV-005; RFNAV-005 adds the separate RFC1/RFD1/RFA1 control/reply plane rather than replacing RFN1.
- RFNAV-002's unsolicited reset root cause is closed as `sys_evt` stack exhaustion under the targeted-scan workload. Keep the configured event-task stack at 4096 bytes unless a later measured workload justifies change.
- RFNAV Tomi-facing telemetry is direct UDP over ECM, ESP `192.168.77.2` to Tomi `192.168.77.1:5515`; do not reroute RFN1 through the Wi-Fi LAN.
- RFNAV control/discovery is direct over ECM using Tomi -> ESP UDP/5516 and ESP -> Tomi UDP/5517.
- Keep telemetry and control socket/formatting work out of `sys_evt` / Wi-Fi callback context.
- RFNAV target state has one canonical ESP-side owner; DISCOVER must not silently replace the active target.
- The original RFNAV-005 pre-repair DISCOVER timeout has no proven exact cause. Preserve that provenance limit.
- Associated-uplink AP selection remains physically unqualified; do not imply that RFNAV-005 qualification tested it.
- RSSI is a received-signal-strength indicator, not a distance measurement.
- GPIO4 remains removed/unconnected and has no RFNAV role.
- Role-aware Tomi boot integration already exists. Do not revive a proposed boot-integration v002.
- Never issue `echo host > /sys/devices/platform/tomtomgo-usbmode/mode`; prior use caused rc=139/kernel Oops behavior.
- Do not connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. See [TomiDock Architecture](docs/architecture/tomidock.md) for the complete safe sequence.

## Open / next work

- **RFNAV-006 scope is not yet adjudicated.** GPS correlation/navigation context is the leading product direction; choose the milestone explicitly before implementation.
- Associated-uplink AP selection may be qualified later as an edge case, but it is not required to reopen RFNAV-005.
- GPS-based directional navigation, mapping, persistence, and BLE remain future work unless separately promoted into scope.
- Deterministic no-GPIO physical-detach fail-close remains formally unproven; historical A8/premature-MOUNT observations remain closed as an unrelated unresolved lifecycle boundary.
- Exact installed provenance for the current `tomidock-netd` and `/etc/rc` remains unbound. The qualified RFNAV ESP and Tomi UI binaries have recorded build identities, but no independent post-install readback of every deployed component is asserted here.
- The exact Tomi VBUS-switch implementation and formal USB self-powered compliance remain undocumented/unestablished.

## Canonical map

- [TT3 “Tomi” device profile](docs/devices/tt3-tomi.md) — device identity and stable hardware facts.
- [TomiDock Architecture](docs/architecture/tomidock.md) — physical/USB/network design, lifecycle, and safety constraints.
- [RF Navigator](docs/architecture/rfnav.md) — RFNAV behavior, protocols, UI, qualification state, review/repair closure, evidence identities, and staged development.
- [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md) — Tomi commands and compatibility.
- [Tomi File Transfer](docs/runbooks/tomi-file-transfer.md) — current transfer procedures.
- [Lab Environment](docs/environment/lab-environment.md) — machine roles, source/build paths, and command contexts.
- [Evidence Index](evidence/INDEX.md) — conventions for locating preserved external evidence.

Detailed reports and raw experiment artifacts remain in `/mnt/d/Codex`; they are evidence stores, not substitutes for current LUCE decisions.
