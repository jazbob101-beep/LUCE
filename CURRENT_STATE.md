# TomTom Revival — Current State

## Current focus

The primary device is TT3 “Tomi,” with TomiDock providing its current USB/network connection. Active development is **RFNAV**, the RF signal-navigator project built on Tomi + TomiDock. **RFNAV-005 remains the current physically qualified RF tracking/control baseline. RFNAV-006 is now the active production-design stage**, explicitly scoped as GPS-correlated directional RF navigation with offline regional map context. The dedicated map feasibility spike passed on physical Tomi: regional `.rfmap` packages render correctly, internal cell boundaries are visually invisible, memory remains bounded, coverage-edge behavior is graceful, repeated relaunch is stable, and map rendering coexists cleanly with RFNAV-005 and TomiDock networking. The current feasibility 80 m/px LOD is intentionally rejected for production because it is both too slow and too visually dense. Routine USB detach-anomaly cycling remains retired.

## Stable operational baseline

- **TomiDock is operational.** Tomi is the USB host; the ESP32-S3 is the CDC-ECM device and Wi-Fi routing endpoint. GPIO4 is unconnected and has no runtime VBUS/session role. Normal ECM, routing/NAPT, and TCP/2323 forwarding remain qualified.
- **RFNAV-005 is the qualified RFNAV baseline.** It retains RFN1 target telemetry on UDP/5515 and adds Tomi-to-ESP RFC1 commands on UDP/5516 plus ESP-to-Tomi RFD1/RFA1 replies on UDP/5517.
- **Qualified ESP application:** 818,848 bytes, SHA-256 `1d8d1bce5e30f0f3ccc5ef8f0e03df1372bbe69729f1b46d8d35cfcadec83a09`.
- **Qualified Tomi UI:** `/mnt/sdcard/opentom/rfnav/ui/nxrfnav-v002`, 20,148 bytes, SHA-256 `865cf76ddbe11ea1a383eea0ef107f23fbe85399d92933257317f28fbc680ba2`, built with `arm-linux-gcc (GCC) 3.3.4` against Nano-X.
- **Qualified merged launcher:** `/mnt/sdcard/opentom/thomas/etc/nxlaunch.cnf`, 217 bytes, SHA-256 `54651633d90457e55425ac43ffe9b8ac6f892d2e79fad823937027ed972d6057`; the RFNAV entry launches the qualified UI without replacing the existing GPS entry.
- **The Tomi diagnostic receiver remains available.** `/mnt/sdcard/opentom/rfnav/bin/rfnav-rx` is 6,512 bytes, SHA-256 `f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6`. It is diagnostic plumbing, not the normal user-facing RFNAV consumer.
- **Offline regional mapping is physically viable.** The feasibility renderer and Houston `.rfmap` packages passed real-device rendering, cell-boundary, memory/relaunch, coverage-edge, and RFNAV/TomiDock coexistence tests. The spike renderer is not itself the production RFNAV-006 UI.
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

RFNAV-006 scope is now adjudicated. It will correlate actual RFN1 `OBS` samples with Tomi GPS state and movement to provide a north-up Signal Rose plus a stability-qualified best-observed signal area. Signal-Rose sectors are historical movement directions associated with improving signal, not claimed transmitter bearings. GPS loss must be nonblocking so ordinary RFNAV-005 tracking remains useful. Initial target changes reset the geographic session; persistence is deferred.

The RFNAV-006 map feasibility spike physically proved the regional offline map direction. Rich 5 m/px first-visible rendering was under 2 seconds by operator observation; the renderer reported 1.623722 s initial base-frame submission. Sparse 5 m/px rendered in 0.569384 s and looked very similar, perhaps cleaner. Rich and sparse 20 m/px were approximately 2.19 s and 2.02 s. Both feasibility 80 m/px views were approximately 10.7–10.9 s and visually over-dense because they still carried roughly 52k–53k points across 55 cells. The production 80 m/px view is therefore an orientation layer, not a compressed street atlas: retain highways/freeways, major arterials, significant water and major place/district context while removing most local-road geometry.

The map qualification also established visually seamless 4 km cell crossings, normal exact-boundary rendering, bounded renderer RSS, clean repeated CLOSE/relaunch behavior, graceful partial-map coverage-edge behavior with no network fallback, and clean coexistence with RFNAV-005 and TomiDock ECM traffic. Final cleanup left neither map nor RFNAV process running and the installed RFNAV UI still matched its qualified hash. The run was conducted while the rig was operating from a USB battery bank after an approximately one-hour functional soak; this is a field-prototype observation, not an electrical-current or capacity qualification.

See [RF Navigator](docs/architecture/rfnav.md) for protocol, UI behavior, RFNAV-006 product design, map qualification details, evidence identities, and limits.

### PowerFlight

A preserved PowerFlight v001.5.3 run archive has six payloads verified against its embedded manifest and no `BLOCKED` events; its Tomi-specific records strongly support a physical Tomi run, but the acquisition path and exact recorder bytes are not bound. Its final recorded state was `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, `BATTERY_RETURN_CHILD_ATTACHED`. See the [power/battery reference](docs/reference/tomi-power-battery.md) for provenance limits.

### nxbattery

Current operator-reported status is that nxbattery V002 is deployed at `/mnt/sdcard/opentom/ui/nxbattery-v002`. The preserved V002 build is 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. With external power detected, suppression of percentage/bar fill is intentional; voltage and raw `CHARGE=<integer>` remain visible.

### Diagnostics

The Phase 4F reduced observer remains diagnostic-only, not production architecture. Preserve it for opportunistic capture only if an unrelated anomaly occurs.

## Closed decisions

- **RFNAV-005 is physically qualified and remains the active RF tracking/control baseline. RFNAV-006 is the active production-design stage, not yet a replacement qualification.**
- `nxrfnav-v002` is the qualified normal user-facing RFNAV consumer; `rfnav-rx` remains diagnostic-only.
- RFNAV-003 RFN1 telemetry remains the tracking transport beneath RFNAV-005; RFNAV-005 adds the separate RFC1/RFD1/RFA1 control/reply plane rather than replacing RFN1.
- RFNAV-002's unsolicited reset root cause is closed as `sys_evt` stack exhaustion under the targeted-scan workload. Keep the configured event-task stack at 4096 bytes unless a later measured workload justifies change.
- RFNAV Tomi-facing telemetry is direct UDP over ECM, ESP `192.168.77.2` to Tomi `192.168.77.1:5515`; do not reroute RFN1 through the Wi-Fi LAN.
- RFNAV control/discovery is direct over ECM using Tomi -> ESP UDP/5516 and ESP -> Tomi UDP/5517.
- Keep telemetry and control socket/formatting work out of `sys_evt` / Wi-Fi callback context.
- RFNAV target state has one canonical ESP-side owner; DISCOVER must not silently replace the active target.
- The original RFNAV-005 pre-repair DISCOVER timeout has no proven exact cause. Preserve that provenance limit.
- Associated-uplink AP selection remains physically unqualified; do not imply that RFNAV-005 qualification tested it.
- RSSI is a received-signal-strength indicator, not a distance measurement or direct transmitter bearing.
- **Offline regional vector mapping is physically viable on Tomi.** Use the proven indexed regional-package/cell/cache architecture as the production starting point rather than restarting map-architecture discovery.
- **The 80 m/px production view is city-orientation context.** It should intentionally discard most local-road geometry; the feasibility package's over-dense approximately 52k-point wide view is not a production target.
- GPS correlation must consume actual RF observations and read the established Tomi GPS state-provider path rather than competing for the GPS UART/FIFO.
- GPIO4 remains removed/unconnected and has no RFNAV role.
- Role-aware Tomi boot integration already exists. Do not revive a proposed boot-integration v002.
- Never issue `echo host > /sys/devices/platform/tomtomgo-usbmode/mode`; prior use caused rc=139/kernel Oops behavior.
- Do not connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. See [TomiDock Architecture](docs/architecture/tomidock.md) for the complete safe sequence.

## Open / next work

- **RFNAV-006 production design is active.** First verify the live Tomi GPS state-provider schema and measure stationary/moving GPS jitter before freezing geographic scoring thresholds.
- Productionize the three map LOD policies. The 5 m/px view may retain useful local detail; 20 m/px should emphasize neighborhood/district structure; 80 m/px should be aggressively simplified to city-orientation features.
- Integrate GPS/RFN1 OBS correlation, Signal Rose, breadcrumb/path context and the stability-qualified best-observed signal area while keeping GPS loss nonblocking.
- Regional packages should support multiple installed metros, package overlap/halo, and no network requirement for ordinary movement inside a loaded metro. Internet access is acceptable for deliberate package acquisition/replacement.
- Associated-uplink AP selection may be qualified later as an edge case, but it is not required to reopen RFNAV-005.
- Geographic-session persistence and BLE remain later work unless separately promoted into scope.
- Deterministic no-GPIO physical-detach fail-close remains formally unproven; historical A8/premature-MOUNT observations remain closed as an unrelated unresolved lifecycle boundary.
- Exact installed provenance for the current `tomidock-netd` and `/etc/rc` remains unbound. The qualified RFNAV ESP and Tomi UI binaries have recorded build identities, but no independent post-install readback of every deployed component is asserted here.
- The exact Tomi VBUS-switch implementation and formal USB self-powered compliance remain undocumented/unestablished.

## Canonical map

- [TT3 “Tomi” device profile](docs/devices/tt3-tomi.md) — device identity and stable hardware facts.
- [TomiDock Architecture](docs/architecture/tomidock.md) — physical/USB/network design, lifecycle, and safety constraints.
- [RF Navigator](docs/architecture/rfnav.md) — RFNAV behavior, protocols, UI, RFNAV-006 production design, map qualification state, evidence identities, and staged development.
- [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md) — Tomi commands and compatibility.
- [Tomi File Transfer](docs/runbooks/tomi-file-transfer.md) — current transfer procedures.
- [Lab Environment](docs/environment/lab-environment.md) — machine roles, source/build paths, and command contexts.
- [Evidence Index](evidence/INDEX.md) — conventions for locating preserved external evidence.

Detailed reports and raw experiment artifacts remain in `/mnt/d/Codex`; they are evidence stores, not substitutes for current LUCE decisions.