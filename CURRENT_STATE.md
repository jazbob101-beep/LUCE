# TomTom Revival — Current State

## Current focus

The primary device is TT3 “Tomi,” with TomiDock providing its current USB/network connection. Active development is **RFNAV**, the RF signal-navigator project built on Tomi + TomiDock. **RFNAV-005 remains the current physically qualified RF tracking/control baseline. RFNAV-006 is the active implementation stage**, explicitly scoped as GPS-correlated directional RF navigation with offline regional map context. The map feasibility spike passed on physical Tomi, and the current GPS state-provider contract plus stationary-jitter behavior have now been physically characterized. The first moving capture was mechanically confounded by a soldered-wire failure. The operator repaired the rig, and the 2026-09-29 moving v002 completes the deferred clean GPS characterization: 228 continuous GPS-quality valid snapshots after acquisition, a healthy provider throughout, and useful stop/resume evidence. The immediate repair blocker is cleared; continuous ECM delivery during the outing was not measured. The GPS/motion and RFN1 OBS core is now implemented, host-tested and built with the pinned legacy ARM compiler as an isolated candidate. Signal Rose/best-area presentation and bounded physical integration follow; conservative schema-1 witness coverage remains an explicit limit. Routine USB detach-anomaly cycling remains retired.

## Stable operational baseline

- **TomiDock is operational on the bench.** Tomi is the USB host; the ESP32-S3 is the CDC-ECM device and Wi-Fi routing endpoint. GPIO4 is unconnected and has no runtime VBUS/session role. Normal ECM, routing/NAPT, and TCP/2323 forwarding remain qualified.
- **The repaired TomiDock rig has completed a bounded mobile outing.** GPS/provider continuity passed; post-return Tomi-Ping was 3/3. Continuous mobile ECM delivery and general mechanical endurance are not claimed. See [TomiDock Architecture](docs/architecture/tomidock.md).
- **RFNAV-005 is the qualified RFNAV baseline.** It retains RFN1 target telemetry on UDP/5515 and adds Tomi-to-ESP RFC1 commands on UDP/5516 plus ESP-to-Tomi RFD1/RFA1 replies on UDP/5517.
- **Qualified ESP application:** 818,848 bytes, SHA-256 `1d8d1bce5e30f0f3ccc5ef8f0e03df1372bbe69729f1b46d8d35cfcadec83a09`.
- **Qualified Tomi UI:** `/mnt/sdcard/opentom/rfnav/ui/nxrfnav-v002`, 20,148 bytes, SHA-256 `865cf76ddbe11ea1a383eea0ef107f23fbe85399d92933257317f28fbc680ba2`, built with `arm-linux-gcc (GCC) 3.3.4` against Nano-X.
- **Qualified merged launcher:** `/mnt/sdcard/opentom/thomas/etc/nxlaunch.cnf`, 217 bytes, SHA-256 `54651633d90457e55425ac43ffe9b8ac6f892d2e79fad823937027ed972d6057`; the RFNAV entry launches the qualified UI without replacing the existing GPS entry.
- **The Tomi diagnostic receiver remains available.** `/mnt/sdcard/opentom/rfnav/bin/rfnav-rx` is 6,512 bytes, SHA-256 `f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6`. It is diagnostic plumbing, not the normal user-facing RFNAV consumer.
- **Offline regional mapping is physically viable.** The feasibility renderer and Houston `.rfmap` packages passed real-device rendering, cell-boundary, memory/relaunch, coverage-edge, and RFNAV/TomiDock coexistence tests. The spike renderer is not itself the production RFNAV-006 UI.
- **The current GPS application-facing source is `/var/run/ttgps.state`.** RFNAV must consume that read-only state file rather than opening `/var/run/gpspipe`, `/dev/gpsdata`, or `/dev/ttySAC1`.
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

RFNAV-006 scope is adjudicated. It will correlate actual RFN1 `OBS` samples with Tomi GPS state and movement to provide a north-up Signal Rose plus a stability-qualified best-observed signal area. Signal-Rose sectors are historical movement directions associated with improving signal, not claimed transmitter bearings. GPS loss must be nonblocking so ordinary RFNAV-005 tracking remains useful. Initial target changes reset the geographic session; persistence is deferred.

The RFNAV-006 map feasibility spike physically proved the regional offline map direction. Rich 5 m/px first-visible rendering was under 2 seconds by operator observation; the renderer reported 1.623722 s initial base-frame submission. Sparse 5 m/px rendered in 0.569384 s and looked very similar, perhaps cleaner. Rich and sparse 20 m/px were approximately 2.19 s and 2.02 s. Both feasibility 80 m/px views were approximately 10.7–10.9 s and visually over-dense because they still carried roughly 52k–53k points across 55 cells. The production 80 m/px view is therefore an orientation layer, not a compressed street atlas: retain highways/freeways, major arterials, significant water and major place/district context while removing most local-road geometry.

The map qualification also established visually seamless 4 km cell crossings, normal exact-boundary rendering, bounded renderer RSS, clean repeated CLOSE/relaunch behavior, graceful partial-map coverage-edge behavior with no network fallback, and clean coexistence with RFNAV-005 and TomiDock ECM traffic. Final cleanup left neither map nor RFNAV process running and the installed RFNAV UI still matched its qualified hash. The run was conducted while the rig was operating from a USB battery bank after an approximately one-hour functional soak; this is a field-prototype observation, not an electrical-current or capacity qualification.

#### RFNAV-006 GPS state and motion characterization

The live GPS path was re-verified on 2026-09-28 as `glgps -> /var/run/gpspipe -> ttgpsd -> /var/run/ttgps.state` plus satellite/raw outputs. `gpspipe` and `glgpsctrl` are named pipes and are not RFNAV application interfaces. The state file publishes provider health/freshness, fix validity, position, speed/course, dilution, satellite counts and UTC in coherent schema-version-1 snapshots. Across both captured runs the state publication rate was approximately 1.666 Hz, with accepted live-fix write-to-last-sentence age never exceeding 1.03 seconds; a two-second stale threshold is the current conservative application candidate.

The clean stationary run contained 300/300 valid fixes over 340.37 seconds. `speed_mps` was exactly 0.000 in all 300 snapshots while the reported coordinates still drifted: the maximum adjacent step was 0.73 m, first-to-last displacement was 16.75 m, and maximum separation between any two stationary positions reached 29.59 m. `course_true_deg` remained fixed at 203.5° throughout. This closes displacement-only motion detection as invalid and establishes that course must not drive Signal-Rose direction while stationary.

The first moving capture is useful but not a clean qualification because the physical TomiDock rig suffered a soldered-wire failure during the attempt. The provider itself stayed alive/stream-open with clean parser counters for the 339.79-second file, but only 146/300 snapshots had valid navigation/position/fix. Valid-fix runs were samples 38-40 and 64-206; samples 1-37, 41-63 and 207-300 were no-fix. Among valid snapshots, 82 reported positive speed; positive walking speeds ranged 0.309-0.926 m/s with median 0.720 m/s. That historical 0.30 m/s candidate is superseded by repaired-rig v002, summarized below.

Repaired-rig v002 lasted 348.02 seconds, with 228 continuous valid GPS-quality snapshots at samples 73-300 and zero provider restart, EOF/reopen or parser error. Positive speed ranged 0.257-1.029 m/s (median 0.617); the long zero-speed interval lasted 128.99 seconds. Initial implementation policy is enter >=0.25 m/s, exit <=0.10 m/s, each after three distinct fresh motion-bearing updates. The first 72 no-fix snapshots retained stale speed 2.624 m/s despite invalid navigation/position/fix. Twenty-two adjacent snapshot pairs reused an RMC timestamp, so provider publication alone must not count as a new motion measurement. Full evidence, provenance and limits are owned by the GPS reference.

Production GPS rules are now preserved in [Tomi GPS and `glgps` Reference](docs/reference/tomi-gps-glgps.md): consume `/var/run/ttgps.state`; separate provider-alive/no-fix from provider failure; pause geographic scoring on invalid fix while leaving RFNAV-005 tracking operational; use reported speed as the primary motion gate; derive movement bearing from accepted geographic displacement only after motion is established; do not use `course_true_deg` while stationary; and initially exclude `fix_quality_name=estimated` from directional-scoring evidence.

The isolated `/home/jazbob/opentom/lab-work/tomi/nxrfnav-v003-rfnav006` core candidate passes 93 new host assertions, actual-provider regressions, deterministic replay of all three GPS logs and the pinned MacBook GCC 3.3.4 build. Moving v002 admits 128/300 synthetic OBS pairs, but conservative raw-RMC witness loss leaves no qualified directional leg; physical RF accuracy is unclaimed. Source/build identities, exact provisional policies and the next integration boundary are owned by RFNAV below. RFNAV-005 remains installed and qualified.

See [RF Navigator](docs/architecture/rfnav.md) for protocol, UI behavior, RFNAV-006 product design, map qualification details, evidence identities, and limits.

### S5.5279 bootloader re-audit

The 2026-09-28 S5.5279 execution-path re-audit re-inspected the earlier TT1 research against the underlying byte-identical TT1/TT3 `SYSTEM` package and corrected one important board-profile attribution. **Austin/type 42 is constructor `0x30b1fad4`, selecting legacy MOVINAND/iNAND SDI and the full-speed USB device controller at `0x52000000`; `0x30b1fe24` is Bergamo/type 43 and owns the previously cited high-speed paths.** The shared FAT/TTBL loader, USB reset-latch admission model and red-X-to-MSC recovery conclusions survive, but controller-specific Austin explanations must use the corrected path.

The generic TTBL loader checks payload integrity with MD5 plus an embedded-key Blowfish tag, but the tag does not authenticate section destination addresses, the final trailer entry or parameter address. Bounded original-instruction tests accepted changed addresses with unchanged valid payload/tag bytes. The ordinary handoff is `0x30b050f4: bx r3`, with `r3` holding the media-supplied trailer entry; no reached public-key vendor-authenticity or entry-range gate was established. USB can deliver executable bytes as ordinary MSC storage blocks for later FAT loading, but no direct USB/UART download-and-go path was found.

The follow-on CMDLINE bounds audit closes the previously unresolved local-copy geometry. The reader accepts declared `CMDLINE.TXT` lengths below 1,024 bytes and normalizes bytes below `0x20` to NUL before an unbounded NUL-terminated copy into a 32-byte `ATAG_CMDLINE` payload. Thirty-one data bytes plus NUL are strictly contained; `S=32` performs an out-of-field zero-on-zero NUL write into the following `ATAG_NONE.size`, effective terminator corruption begins at `S=33`, saved `r4` at `S=40`, saved `lr` at `S=44`, and the caller frame at `S=48`. The exported `0xdc`-byte Linux parameter block includes the `ATAG_NONE` area but excludes the saved-register/caller-frame bytes. The normal tested handoff reaches `bx r3` before those saved registers are restored, so the audit establishes exact memory/parameter bounds but does not establish a post-handoff control-flow consequence or exploit path.

A separate S3C2412 retained-state branch uses INFORM1 as a resume target and transfers at `0x0000a550: mov pc,r0`; source correlation identifies this as the intended suspend/resume contract rather than an external loader. Successful `LTSYSTEM` loading requests reset rather than normal trailer handoff, and a SYSTEM first-section size other than `0x40000` selects the update attempt rather than universal rejection. Exact installed NOR identity, especially the protected low prefix/reset path, remains unresolved. Detailed ownership is in [Tomi Bootloader Image Reference](docs/reference/tomi-bootloader-image.md) and [Tomi Boot Chain](docs/architecture/tomi-boot-chain.md).

### PowerFlight

A preserved PowerFlight v001.5.3 run archive has six payloads verified against its embedded manifest and no `BLOCKED` events; its Tomi-specific records strongly support a physical Tomi run, but the acquisition path and exact recorder bytes are not bound. Its final recorded state was `VBUS=0`, `CHARGE=0`, `CHILD=1`, `USB_STATE_HOST`, `last_error=0x0`, `BATTERY_RETURN_CHILD_ATTACHED`. See the [power/battery reference](docs/reference/tomi-power-battery.md) for provenance limits.

### nxbattery

Current operator-reported status is that nxbattery V002 is deployed at `/mnt/sdcard/opentom/ui/nxbattery-v002`. The preserved V002 build is 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. With external power detected, suppression of percentage/bar fill is intentional; voltage and raw `CHARGE=<integer>` remain visible.

### Diagnostics

The Phase 4F reduced observer remains diagnostic-only, not production architecture. Preserve it for opportunistic capture only if an unrelated anomaly occurs.

## Closed decisions

- **RFNAV-005 is physically qualified and remains the active RF tracking/control baseline. RFNAV-006 remains production design, not yet a replacement qualification.**
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
- GPS correlation consumes actual RF observations and reads `/var/run/ttgps.state`; do not compete for the GPS UART/FIFO.
- Displacement-only movement detection is rejected. Bench-stationary GPS drift reached nearly 30 m across the run while reported speed remained zero.
- `course_true_deg` is not a stationary movement-direction source. Use speed as the primary motion gate and derive movement direction from accepted position displacement after motion is established.
- Initial v002-informed motion policy is enter >=0.25 m/s, exit <=0.10 m/s, with three distinct fresh motion-bearing updates per transition. It requires implementation/replay validation and is not a universal receiver guarantee.
- GPS provider-alive/no-fix is a normal nonblocking RFNAV state: geographic scoring pauses, ordinary RFNAV-005 tracking remains usable.
- S5.5279 Austin/type42 uses the legacy MOVINAND/iNAND SDI and full-speed USB-device path; the previously cited HSMOVINAND/high-speed constructor is Bergamo/type43 and must not be used as Austin evidence.
- TTBL payload integrity does not authenticate load destinations, trailer entry or parameter address; the reached generic handoff contains no public-key/vendor-authenticity or entry-range gate.
- S5.5279 `CMDLINE.TXT` has a strict contained limit of 31 data bytes plus NUL in the 32-byte ATAG payload; effective `ATAG_NONE` corruption begins at `S=33`, saved `r4` at `S=40`, saved `lr` at `S=44`, and the caller frame at `S=48`. These are exact package-level bounds, not a proven post-handoff control-flow consequence.
- S5.5279 USB recovery is device-side MSC/block storage for later FAT/TTBL loading; no direct USB/UART download-and-go service is established.
- The retained-INFORM1 branch is a separate resume control transfer, not evidence of an external downloader.
- GPIO4 remains removed/unconnected and has no RFNAV role.
- Role-aware Tomi boot integration already exists. Do not revive a proposed boot-integration v002.
- Never issue `echo host > /sys/devices/platform/tomtomgo-usbmode/mode`; prior use caused rc=139/kernel Oops behavior.
- Do not connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. See [TomiDock Architecture](docs/architecture/tomidock.md) for the complete safe sequence.

## Open / next work

- Integrate visible GPS status and Signal Rose/best-area presentation with the implemented, host-tested and ARM-built RFNAV-006 core candidate. Measure live witness coverage and UI/runtime behavior before promotion; another GPS-only walk is not a prerequisite.
- Preserve the repaired-rig bounded mobile result and its continuous-ECM evidence limit. Full RFNAV-006 field qualification follows implementation.
- Productionize the three map LOD policies. The 5 m/px view may retain useful local detail; 20 m/px should emphasize neighborhood/district structure; 80 m/px should be aggressively simplified to city-orientation features.
- Integrate GPS/RFN1 OBS correlation, Signal Rose, breadcrumb/path context and the stability-qualified best-observed signal area while keeping GPS loss nonblocking.
- Regional packages should support multiple installed metros, package overlap/halo, and no network requirement for ordinary movement inside a loaded metro. Internet access is acceptable for deliberate package acquisition/replacement.
- Associated-uplink AP selection may be qualified later as an edge case, but it is not required to reopen RFNAV-005.
- Geographic-session persistence and BLE remain later work unless separately promoted into scope.
- Deterministic no-GPIO physical-detach fail-close remains formally unproven; historical A8/premature-MOUNT observations remain closed as an unrelated unresolved lifecycle boundary.
- Exact installed provenance for the current `tomidock-netd` and `/etc/rc` remains unbound. The qualified RFNAV ESP and Tomi UI binaries have recorded build identities, but no independent post-install readback of every deployed component is asserted here.
- The exact Tomi VBUS-switch implementation and formal USB self-powered compliance remain undocumented/unestablished.
- Exact installed S5.5279 NOR identity, including the protected low 32 KiB/reset-prefix route, remains unresolved. A future read-only capture is conditional on an already-established safe access path; no new flash/update experiment is required merely to re-prove package-level findings.

## Canonical map

- [TT3 “Tomi” device profile](docs/devices/tt3-tomi.md) — device identity and stable hardware facts.
- [Tomi Boot Chain](docs/architecture/tomi-boot-chain.md) — S5.5279 stage ordering, corrected Austin profile path, CMDLINE/ATAG construction bounds, TTBL handoff and retained-RAM resume boundary.
- [Tomi Bootloader Image Reference](docs/reference/tomi-bootloader-image.md) — S5.5279 package identity, exact CMDLINE copy bounds, TTBL trust model, USB/MMC path, updater and control-transfer details.
- [TomiDock Architecture](docs/architecture/tomidock.md) — physical/USB/network design, portable-power observation, mechanical field limit, lifecycle, and safety constraints.
- [RF Navigator](docs/architecture/rfnav.md) — RFNAV behavior, protocols, UI, RFNAV-006 production design, map qualification state, evidence identities, and staged development.
- [Tomi GPS and `glgps`](docs/reference/tomi-gps-glgps.md) — GPS stack, current `ttgpsd` state-provider contract, stationary/moving characterization, and RFNAV-006 motion-gating rules.
- [Tomi Runtime ABI](docs/reference/tomi-runtime-abi.md) — Tomi commands and compatibility.
- [Tomi File Transfer](docs/runbooks/tomi-file-transfer.md) — current transfer procedures.
- [Lab Environment](docs/environment/lab-environment.md) — machine roles, source/build paths, and command contexts.
- [Evidence Index](evidence/INDEX.md) — conventions for locating preserved external evidence.

Detailed reports and raw experiment artifacts remain in `/mnt/d/Codex`; they are evidence stores, not substitutes for current LUCE decisions.
