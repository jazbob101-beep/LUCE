# RF Navigator (RFNAV)

## Purpose

RFNAV repurposes TT3 “Tomi” and TomiDock as a geographically aware RF signal navigator. Tomi provides GPS, touchscreen, Linux, and battery operation; the ESP32-S3 in TomiDock provides Wi-Fi/BLE radio capability. The intended product direction is to discover authorized nearby transmitters, select a target, measure signal strength repeatedly, and correlate that telemetry with Tomi-side state so Tomi can eventually help navigate toward a signal source.

This document is the canonical owner for RFNAV behavior, qualification state, protocol, UI behavior, mapping/navigation context, and staged development. TomiDock USB/network architecture, lifecycle behavior, and electrical safety remain owned by [TomiDock Architecture](tomidock.md).

## Current qualified baseline

The current qualified RFNAV baseline is **RFNAV-005**.

RFNAV-005 retains the physically qualified RFNAV-003 RFN1 telemetry architecture and the RFNAV-004 live signal model, then adds desktop integration, operator-driven Wi-Fi discovery, paged touchscreen target selection, TRACK / Change Target workflow, control-plane recovery, and stricter protocol validation.

Qualified ESP application image:

```text
size: 818848 bytes
SHA-256: 1d8d1bce5e30f0f3ccc5ef8f0e03df1372bbe69729f1b46d8d35cfcadec83a09
```

ESP source:

```text
/home/jazbob/opentom/lab-work/esp32-s3/tomi-s3-native-ecm-v002-routing-v001-nogpio-phase3-rfnav-v005
```

Qualified Tomi UI:

```text
/mnt/sdcard/opentom/rfnav/ui/nxrfnav-v002
size: 20148 bytes
SHA-256: 865cf76ddbe11ea1a383eea0ef107f23fbe85399d92933257317f28fbc680ba2
```

Tomi source:

```text
/home/jazbob/opentom/lab-work/tomi/nxrfnav-v002
```

The qualified UI was built with the pinned legacy Tomi toolchain, `arm-linux-gcc (GCC) 3.3.4`, in the Bookworm chroot and links only `libnano-X.so` and `libc.so.6` dynamically. The only reported target-build warning is the known historical Nano-X `index` shadow warning.

The live desktop launcher was reconciled against the installed Tomi configuration rather than replacing it wholesale. Qualified merged launcher identity:

```text
/mnt/sdcard/opentom/thomas/etc/nxlaunch.cnf
size: 217 bytes
SHA-256: 54651633d90457e55425ac43ffe9b8ac6f892d2e79fad823937027ed972d6057
```

RFNAV launcher entry:

```text
RFNAV /mnt/sdcard/opentom/rfnav/ui/rfnav.pgm /mnt/sdcard/opentom/rfnav/ui/nxrfnav-v002
```

RFNAV icon identity:

```text
/mnt/sdcard/opentom/rfnav/ui/rfnav.pgm
size: 3966 bytes
SHA-256: bc857525b11dd780a522a0845d2d3b6337ade42c50f8734914e952fe08b2ba55
```

The earlier diagnostic receiver remains available at:

```text
/mnt/sdcard/opentom/rfnav/bin/rfnav-rx
size: 6512 bytes
SHA-256: f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6
```

`rfnav-rx` remains diagnostic plumbing. `nxrfnav-v002` is the qualified user-facing RFNAV application.

## RFNAV-002 behavior retained

RFNAV-005 retains the established RFNAV focused-scan behavior beneath the operator-selected target workflow:

1. repeated nonblocking scans filtered to the selected BSSID and channel;
2. a nominal 2-second quiet interval between target scans;
3. target RSSI reporting when observed;
4. rediscovery after five consecutive valid target misses;
5. target invalidation and fresh discovery when Wi-Fi state changes.

Target scans retain active maximum dwell 120 ms and home-channel dwell 30 ms. Wi-Fi power save remains ESP-IDF 5.5.5 `WIFI_PS_MIN_MODEM`. RFNAV does not use GPIO4 and does not alter TomiDock USB lifecycle policy.

## RFNAV-002.1 / 002.2 stability closure

RFNAV-002.1 reproduced the unsolicited restart three times with Tomi completely absent. Each failure reported a `sys_evt` stack overflow after `TARGET_SCAN_REQUEST` and before `TARGET_SCAN_STARTED`, at approximately 332.5 s, 204.8 s, and 192.1 s uptime.

For that build:

```text
CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE = 2304
TASK_EXTRA_STACK_SIZE                   = 512
old effective sys_evt stack             = 2816 bytes
```

RFNAV-002.2 raised the configured event-task stack to 4096 bytes, 4608 effective bytes in this ESP-IDF 5.5.5 build, and added `sys_evt` high-water telemetry.

Physical instrumentation measured a worst-case `sys_evt` high-water mark of 1648 free bytes:

```text
4608 total - 1648 free = approximately 2960 bytes maximum observed use
```

That observed use exceeds the old 2816-byte allocation by approximately 144 bytes. RFNAV-002.2 then passed standalone and full TomiDock physical qualification. This closes the RFNAV-002.1 reset root cause as event-task stack exhaustion under the repeated targeted-scan workload.

RFNAV-003, RFNAV-004, and RFNAV-005 retain `CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE=4096` and the `SYS_EVT_HEALTH` diagnostic.

## RFNAV-003 Tomi telemetry architecture

RFNAV-003 added a unidirectional telemetry path directly across the existing ECM subnet:

```text
ESP ECM peer 192.168.77.2
        |
        | UDP/5515 RFN1 telemetry
        v
Tomi usb0 192.168.77.1
```

The existing LAN-side UDP/5514 diagnostics remain separate and unchanged.

### Wire format

Each Tomi-bound datagram contains one newline-terminated ASCII record:

```text
RFN1 seq=<u32> epoch=<u32> event=<SELECT|OBS|MISS|REDISCOVER|INVALIDATE> target=<12hex|none> ch=<0..14> rssi=<-128..127|na> misses=<0..255>
```

SSID is intentionally not carried. Target identity is the BSSID encoded as twelve hexadecimal digits without separators. Do not reproduce bench-specific BSSIDs in public canonical documentation.

### ESP execution model

Telemetry publication is subordinate to RFNAV scanning and TomiDock operation:

- RFNAV worker code copies bounded telemetry records into a 16-entry queue using zero wait;
- Wi-Fi / system-event callback context does not create sockets, format datagrams, or call `sendto()`;
- a separate priority-1 sender task with a 4096-byte stack owns formatting and nonblocking UDP transmission;
- destination is Tomi at `192.168.77.1:5515`;
- queue overflow, ECM-unready state, stale records, retry backoff, and send errors are counted rather than allowed to block or alter RFNAV/TomiDock control flow;
- telemetry queued while Tomi is absent is not allowed to grow without bound.

Low-frequency `RFNAV_TX_HEALTH` telemetry reports sender stack margin and counters alongside the existing `RFNAV_HEALTH` and `SYS_EVT_HEALTH` diagnostics.

### Tomi diagnostic receiver

`rfnav-rx` is a small Tomi-native UDP receiver built with the pinned GCC 3.3.4 target toolchain. It binds UDP/5515, validates the `RFN1` prefix and bounded datagram length, records the sender IPv4 address, tracks receive count and sequence continuity, and reports gaps, rewinds, malformed records, and oversized records.

## RFNAV-003 physical qualification

RFNAV-003 was tested in the normal TomiDock topology with Tomi attached as USB host, CDC-ECM mounted, routing/NAPT and TCP/2323 active, the Tomi-native `rfnav-rx` receiver bound to UDP/5515, continuous LAN ping running, LAN-side UDP diagnostics active, and repeated RFNAV target scans continuing.

The receiver captured **484 consecutive RFN1 datagrams**, sequence 106 through 589, all from `192.168.77.2`, with zero gaps, rewinds, source changes, malformed records, or oversized records. This physically proved direct ESP-to-Tomi RFN1 delivery over the ECM link.

During the same qualification:

- `RFNAV_TX_HEALTH` reached at least `sent=506`;
- `qdrop=0`, `send_err=0`, `stale_drop=0`, and `backoff_drop=0` throughout while Tomi was present;
- `ecm_drop` stabilized at 96 after Tomi became available;
- sender-task stack high-water mark settled at 2104 free bytes;
- `sys_evt` high-water mark remained 1648 free bytes;
- RFNAV worker high-water mark remained 1148 free bytes;
- minimum-ever free heap reached 195732 bytes and then remained stable in retained final health samples;
- 538/538 target scans completed with `status=0`;
- zero RFNAV rediscoveries occurred;
- 1,828/1,828 LAN ping samples succeeded;
- no panic, stack overflow, reboot, Wi-Fi disconnect, USB suspend, or USB unmount was observed.

RFNAV-003 therefore passed physical qualification as the Tomi-facing telemetry transport baseline.

## RFNAV-004 Tomi live signal display

RFNAV-004 is Tomi-side only. The ESP firmware and RFN1 protocol remain the qualified RFNAV-003 baseline unchanged.

The user-facing application is `nxrfnav-v001`. It binds UDP/5515 directly and is an alternate consumer to `rfnav-rx`; the diagnostic receiver should not hold the port while the UI runs.

### Display behavior

The 480x272 Nano-X display presents:

- abbreviated target identity;
- current Wi-Fi channel;
- large raw RSSI in dBm;
- a progressive signal-strength gauge using smoothed RSSI;
- stronger / steady / weaker trend classification;
- current target state;
- live / stale / no-telemetry freshness state.

The gauge clamps approximately `-90 dBm` through `-30 dBm`. Smoothing uses an integer EMA with alpha 0.5. Trend compares a short four-reading smoothed history with a +/-2 dB deadband. Freshness is classified as LIVE through 5 seconds, STALE through 10 seconds, and NO TELEMETRY after that.

The UI parses the qualified RFN1 events directly. SELECT establishes a target and resets signal history; OBS updates RSSI, smoothing, gauge, trend, and freshness; MISS suppresses current RSSI while briefly retaining the prior gauge state; REDISCOVER clears current signal/trend interpretation; INVALIDATE clears the active target and signal history.

The application uses a nonblocking UDP socket plus a timed Nano-X event wait and does not require per-packet dynamic allocation or a background shell pipeline.

### Target-toolchain compatibility cleanup

The initial target build emitted an implicit `snprintf` declaration warning under GCC 3.3.4. The source was corrected by defining:

```c
#define _ISOC99_SOURCE 1
```

before the existing POSIX feature macro and all includes. The pinned glibc 2.3 headers then exposed the C99 `snprintf` declaration correctly.

The corrected canonical rebuild emitted no `snprintf` warning. The only remaining warning came from the historical Nano-X header's `index` declaration shadowing a global declaration.

The corrected source produced the same stripped executable bytes as the earlier build:

```text
size: 13708 bytes
SHA-256: a7947016ae034199dcbab1240635f503cf2f003226c04e19ca0ccabda90ef365
```

This confirms the feature-test correction changed declaration visibility without changing generated target machine code.

## RFNAV-004 physical qualification

RFNAV-004 was deployed to Tomi and launched against the already-qualified RFNAV-003 live telemetry stream.

The initial live-screen smoke test showed the Nano-X UI rendering correctly and consuming current RFN1 OBS telemetry. Target identity, channel, numeric RSSI, signal-strength gauge, target state, trend state, and freshness state populated on the physical Tomi display.

The application then remained running continuously for approximately **12 hours**. This endurance result is operator-observed rather than backed by a continuous 12-hour packet/diagnostic log. During that interval:

- no functional instability was observed;
- the signal-strength gauge moved in unison with the displayed RSSI as signal conditions changed;
- the weak-to-strong presentation behaved correctly across observed signal changes;
- no UI or RFNAV functional issue was reported.

This extended run materially exceeds the original short smoke/soak intent and qualifies `nxrfnav-v001` as the RFNAV-004 user-facing interface for the observed normal target-tracking workload.

Host-side qualification separately covers bounded RFN1 parsing, all five protocol event types, malformed/oversized input, sequence continuity/wrap/gap/rewind handling, source and epoch changes, stale/no-telemetry timeouts, trend behavior, and gauge clamping. Those host tests complement but do not imply that every exceptional RFN1 state was deliberately forced during the 12-hour physical run.

## RFNAV-005 discovery and target-selection architecture

RFNAV-005 adds a bidirectional product-control plane on the existing ECM subnet while retaining RFN1 telemetry unchanged:

```text
Tomi 192.168.77.1  -> ESP 192.168.77.2 UDP/5516   RFC1 commands
ESP  192.168.77.2  -> Tomi 192.168.77.1 UDP/5517   RFD1 / RFA1 replies
ESP  192.168.77.2  -> Tomi 192.168.77.1 UDP/5515   RFN1 telemetry unchanged
```

Control and discovery records are newline-terminated bounded ASCII records:

```text
RFC1 req=<u32> cmd=DISCOVER
RFC1 req=<u32> cmd=SELECT target=<12hex>

RFD1 req=<u32> idx=<0..47> total=<0..48> target=<12hex> ch=<1..14> rssi=<-128..127> assoc=<0|1> ssidhex=<hex|none>

RFA1 req=<u32> cmd=DISCOVER status=OK total=<0..48> truncated=<0|1>
RFA1 req=<u32> cmd=SELECT status=OK target=<12hex>
```

Discovery retains at most 48 APs, orders them strongest first with a deterministic BSSID tie-break, and carries SSID as bounded hexadecimal data rather than trusting display text from the radio result. SELECT is authorized only against a sufficiently recent discovery result from the same Wi-Fi epoch. Tomi retries a request with the same request ID; the ESP replay path is bounded and epoch-aware.

The control service has one socket owner and a bounded reply queue. Transient ECM unavailability or socket setup failure does not permanently kill the control plane. Low-frequency `RFNAV_CTRL: HEALTH` telemetry exposes readiness, receive/accept/reject counts, queue drops, sends, socket/send failures, reply drops, and stack margin.

The RFNAV worker remains the canonical owner of target state. Manual DISCOVER does not silently replace the active target. Manual SELECT updates that canonical target and uses the existing RFN1 SELECT event to synchronize the live TRACK display.

### RFNAV-005 Tomi UX

`nxrfnav-v002` integrates with the normal Nano-X desktop and has two primary screens:

- **DISCOVER:** paged nearby-AP list with sanitized SSID, abbreviated target identity, channel, RSSI/strength, associated-AP indication, PREV/NEXT navigation, RESCAN, and CLOSE;
- **TRACK:** selected target identity, channel, raw RSSI, progressive gauge, trend, freshness, Change Target, and CLOSE.

The UI preserves the RFNAV-004 signal model and freshness behavior while adding strict RFC1/RFD1/RFA1 numeric parsing, incomplete-discovery discard, request/reply matching, and the target-selection state machine.

A second application instance is rejected rather than competing for the RFN1 UDP/5515 consumer role. CLOSE releases the application resources so the diagnostic receiver or a later RFNAV launch can bind the port normally.

## RFNAV-005 review and repair closure

The initial RFNAV-005 candidate produced the intended desktop and visual UX but the first physical DISCOVER attempt timed out. A later independent Astra review was deliberately asked to audit the implementation rather than assume the leading diagnosis was correct.

That review **rejected the proposed early-bind/address-absence explanation as stated**. In the actual startup order the static ECM interface is initialized before RFNAV startup, and the pinned lwIP UDP bind implementation does not require the requested local address to be owned by a live interface in the way the hypothesis assumed.

The review did reproduce and repair several real defects:

- transient control socket/setup failure could permanently terminate the control task;
- control initialization failure could bypass the focused-scan delay;
- replay state could reference stale epochs or discovery data overwritten by a failed scan;
- Tomi reply parsing accepted overflowing request IDs, allowing an out-of-range decimal value to alias a smaller `u32` request ID;
- an incomplete discovery list could remain selectable after timeout/error.

Regression tests exercise production control-task and worker logic, including delayed ECM availability, injected setup failures, ECM loss/recovery, persistent errors, discovery while tracking, replay, selection, expiry, malformed input, and strict Tomi protocol parsing. The original source snapshots fail the new regressions; the repaired source passes the expanded host tests, ASan/UBSan checks, GNU89 syntax checks, and an ESP-IDF 5.5.5 full clean build.

The exact cause of the **original pre-repair physical DISCOVER timeout remains unresolved** because the original run did not preserve the listener-status and request/reply discriminator data needed to bind that failure to one repaired defect. Qualification therefore rests on the repaired candidate's positive physical results, not on a retrospective causal claim about the first timeout.

## RFNAV-005 physical qualification

RFNAV-005 passed physical qualification on 2026-09-27 with the repaired ESP and Tomi artifacts identified above.

The startup qualification deliberately booted the ESP before Tomi/ECM was present, then attached Tomi later. The resulting product path became operational and DISCOVER populated a real nearby-AP list. The first operator-visible discovery produced six pages of APs; a later RESCAN produced five pages because at least one previously visible AP was no longer present, which is normal radio-environment variation rather than stale-list behavior.

Operator qualification established:

- PREV/NEXT worked across the multi-page list; button size and placement were reported fully usable on the physical resistive touchscreen;
- RESCAN repopulated the AP set cleanly;
- a non-associated AP could be selected and transitioned to TRACK with sane target/channel/RSSI data;
- gauge and reading changed with live signal conditions and did not become stale during observation;
- Change Target returned to discovery and a second non-associated AP selected and tracked correctly;
- CLOSE returned to the Nano-X desktop as designed;
- RFNAV relaunched successfully and a second complete examination found no issue;
- the duplicate-instance guard passed;
- UDP/5515 ownership cleanup passed, allowing the diagnostic receiver to bind after RFNAV closed;
- one bounded ECM loss/recovery test passed without reviving broad historical detach-anomaly cycling.

The instrumented recovery capture showed the intended lifecycle explicitly: RFNAV control entered `waiting_for_ecm`; ECM reported suspended/unready and routing/NAPT/TCP2323 disabled; `RFNAV_CTRL: HEALTH` reported `ready=0`; the control service later reported `ready local=192.168.77.2:5516`; ECM returned ready in a new generation; and routing/NAPT/TCP2323 returned. Manual DISCOVER then succeeded again after recovery, with later captures retaining 12 and 14 APs respectively. Control health after recovery remained clean, with accepted requests and zero recorded rejects, queue drops, send errors, socket errors, or reply drops in the preserved sample.

Associated-uplink AP selection was intentionally **not** exercised during this qualification. The discovery representation includes the associated AP, but selecting the currently associated uplink remains an unqualified edge case and is not part of the RFNAV-005 PASS claim.

## RFNAV-006 adjudicated scope and production design

RFNAV-006 is now explicitly scoped as **GPS-correlated directional RF navigation with offline regional map context**. The goal is not to claim a transmitter bearing from RSSI. It is to combine repeated RF observations with Tomi's movement and location history so the operator can see which movement directions and visited areas have correlated with stronger signal.

The product model is:

- a north-up **Signal Rose** whose warm sectors represent movement directions that historically correlated with improving signal, not a measured bearing to the transmitter;
- a stability-qualified **best observed signal patch/area**, not a point estimate of transmitter location;
- ordinary RFNAV-005 tracking remains usable when GPS is unavailable or stale;
- geographic scoring consumes actual RFN1 `OBS` samples only; cached DISCOVER/SELECT RSSI is not geographic evidence;
- target change resets the active geographic session initially; cross-session persistence is deferred until separately designed;
- RFNAV reads the existing Tomi GPS state-provider output rather than competing for the GPS UART/FIFO. The exact live provider schema and stationary/moving jitter envelope must be physically verified before production scoring constants are frozen.

Maps are **context, not routing**. RFNAV-006 does not attempt turn-by-turn navigation.

### Regional offline package model

The production direction is regional offline packages such as `houston.rfmap` and `austin.rfmap`. Multiple packages may coexist on Tomi. Internet access is acceptable when deliberately obtaining or replacing a regional package, but ordinary movement within a metro area must not require network access. Internal cells remain an implementation detail and must stay invisible to the operator. Adjacent regional packages should include sufficient overlap/halo to avoid brittle edge behavior.

The physically tested package architecture used 4 km indexed cells, three LODs, local integer geometry, host-side preprocessing/triangulation, bounded cell loading, and a roughly 1 MiB geometry-cache ceiling. That architecture is accepted as the starting production design because the physical spike demonstrated correct rendering, bounded memory behavior, invisible cell transitions, graceful coverage-edge behavior, and clean coexistence with RFNAV/TomiDock.

### Production LOD intent

The three map scales have different jobs and should not carry the same feature density:

- **5 m/px:** local/street-level context. Local roads are useful here.
- **20 m/px:** neighborhood/district context. Prefer collectors, arterials, significant local structure, water, and useful labels over exhaustive street density.
- **80 m/px:** city-orientation context: “what part of town am I in?” This scale should emphasize freeways/highways, major arterials, significant water and major place/district context. Most local-road geometry should be omitted.

The feasibility package's 80 m/px LOD is rejected as a production feature set. It rendered roughly 52,000 points across 55 cells and was both approximately 10.7–10.9 seconds per redraw and visually over-dense to the point of poor usability. Production work should reduce wide-view geometry aggressively, with an initial target on the order of removing roughly 80% of the currently visible road detail, then validate readability and redraw latency on Tomi. The objective is not to optimize drawing unwanted roads faster; it is to stop drawing them.

## RFNAV map feasibility spike and physical qualification

A dedicated feasibility work unit at `/mnt/d/Codex/TT3/rfnav-map-feasibility-20260927/` evaluated whether Tomi can support an offline regional vector-map layer without depending on the proprietary TomTom renderer. Host work produced an OSM-derived `.rfmap` package format, sparse and rich Houston packages, a Nano-X target renderer, previews, bounded loader/cache behavior, and a physical qualification plan.

Frozen deployed artifacts:

```text
nxrfmap-spike
size: 24356 bytes
SHA-256: 3ac5d68f405b47a0078932ce4f6581666f8d1b1f31628eb30edf81259c085b81

houston-sparse.rfmap
size: 17133968 bytes
SHA-256: a5225c3789ae6b277c5aad3750908729c664607072befc3878f6d1c1b16539cf

houston-rich.rfmap
size: 38787714 bytes
SHA-256: 802e38b044cbcbd8ab42256561a1145b2b998f0342c3ba2f1a6a3525a7752a9f

NOTICE.txt
SHA-256: 65e9112c1d4e10339edc5a794dcc3a53e014186d57a8980a02baa625dfee26cb
```

The physical run on 2026-09-27/28 established:

- rich Houston 5 m/px first-visible map in under 2 seconds by operator observation; renderer diagnostics reported package open `0.264510 s`, initial base-frame submission `1.623722 s`, and first overlay submission `0.057496 s`;
- the 5 m/px rich view was recognizable and period-appropriate on the real 480x272 display, with roads, water, readable major labels, synthetic overlays, attribution and no visible corruption;
- overlay-only updates were normally about 36–40 ms and did not force repeated base-map redraws;
- renderer process RSS was only 868 kB in the initial open state. After wider interaction it was 1,652 kB; after the tenth relaunch at 20 m/px it was 824 kB. The low `MemFree` values observed while maps were active tracked a large reclaimable filesystem page cache rather than runaway renderer RSS;
- repeated CLOSE/relaunch operation completed cleanly, final CLOSE removed the process, and memory recovered without evidence of a leak;
- 4 km internal cell crossings were visually undetectable. An exact known cell-boundary center also rendered normally with no missing geometry, flicker or glitching;
- the coverage-edge test produced a partial map plus `COVERAGE EDGE: no network fallback`, remained responsive, and did not crash;
- RFNAV-005 and the map renderer coexisted cleanly. DISCOVER/TRACK/Change Target/CLOSE behaved normally while the map remained open; TomiDock ECM ping remained 5/5 with 0% loss, and closing RFNAV left the map process alive as intended;
- final cleanup left neither map nor RFNAV process running; final TomiDock ping was 5/5 with 0% loss at 2.389 ms average, and the installed RFNAV UI still matched qualified SHA-256 `865cf76ddbe11ea1a383eea0ef107f23fbe85399d92933257317f28fbc680ba2`;
- the entire map qualification occurred while the TomiDock rig was operating from a USB battery bank that had already completed an approximately one-hour functional soak without sleeping or disrupting Tomi/TomiDock operation. This is a field-prototype power observation, not an electrical-current or energy-capacity qualification.

### Measured LOD behavior

The physical renderer timing showed a clear feature-density boundary:

```text
                 rich                 sparse
5 m/px      1.623722 s           0.569384 s
20 m/px     2.194572 s           2.022638 s
80 m/px    10.893992 s          10.733431 s
```

At 5 m/px, sparse reduced the sampled point count from 23,383 to 5,138 and was materially faster while the operator judged it visually very similar and perhaps cleaner. At 80 m/px, rich and sparse still carried about 53,065 and 52,089 points respectively across 55 cells, so both remained approximately 10.7–10.9 seconds and visually over-dense. The second rich 80 m/px redraw remained approximately 10.76 seconds despite only 8,320 bytes of new I/O, so storage/cache miss latency is not the primary wide-view bottleneck. The production fix is a much more aggressive wide-view LOD rather than a different package architecture.

The feasibility question is therefore closed positively: **offline regional vector maps are physically viable on Tomi**. The map spike is not itself the RFNAV-006 production UI, and the current 80 m/px feature set is explicitly not production-qualified.

## Current design conclusions

- **RFNAV-005 remains the current physically qualified RF tracking/control baseline.** RFNAV-006 is the active production-design stage built on that baseline, not a replacement qualification yet.
- `nxrfnav-v002` is the current qualified user-facing RFNAV application.
- RFNAV discovery, paged target selection, live tracking, target changes, close/relaunch, single-instance handling, and UDP/5515 cleanup all work on the physical Tomi/TomiDock system.
- The RFNAV control plane recovers from a bounded real ECM loss/recovery cycle without permanently dying.
- The original pre-repair DISCOVER timeout's exact cause remains unresolved; do not rewrite history as though the later review proved one specific causal defect.
- The associated-uplink AP selection path remains physically untested and is outside the current qualification claim.
- The RFNAV-002 reset root cause remains closed as insufficient `sys_evt` stack; RFNAV-005 retains the 4096-byte configured event-task allocation.
- RFNAV telemetry remains direct UDP over ECM, ESP `192.168.77.2` to Tomi `192.168.77.1:5515`; the product path is not routed through the Wi-Fi LAN.
- Control/discovery traffic remains direct over ECM on UDP/5516 and UDP/5517.
- Socket/formatting work remains outside `sys_evt` / Wi-Fi callback context.
- RSSI is received signal strength, not physical distance or a direct transmitter bearing.
- Offline regional mapping is physically viable on Tomi; the proven package/cell/cache architecture is the production starting point.
- Wide-area 80 m/px mapping is an orientation view and must intentionally omit most local-road geometry rather than preserving the feasibility package's over-dense LOD.

## Safety and operating constraints

RFNAV inherits all TomiDock electrical and USB safety constraints. Never connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. Use the isolation sequence documented in [TomiDock Architecture](tomidock.md).

Do not reintroduce GPIO4 monitoring or gating. GPIO4 is unconnected and has no runtime role in the current TomiDock design.

Do not perform routine USB detach-anomaly cycling as an RFNAV qualification step. RFNAV-005's one bounded ECM recovery check does not reopen the retired detach-anomaly investigation. Preserve an unsolicited lifecycle anomaly if one occurs during otherwise authorized work, but do not turn ordinary RFNAV work into broad lifecycle cycling.

## Development stages

Completed:

- **RFNAV-001:** prove full background Wi-Fi discovery scanning can coexist with TomiDock networking.
- **RFNAV-002:** prove discovery -> deterministic target selection -> repeated focused RSSI sampling while TomiDock remains operational.
- **RFNAV-002.1:** instrument and identify the unsolicited reset as `sys_evt` stack overflow.
- **RFNAV-002.2:** increase and instrument `sys_evt` stack; pass standalone and full-stack physical qualification.
- **RFNAV-003:** establish and physically qualify direct Tomi-facing RFN1 telemetry over the existing ECM link, including a Tomi-native diagnostic receiver.
- **RFNAV-004:** build and physically qualify the first Tomi-native live RF signal display, including extended approximately 12-hour continuous operation.
- **RFNAV-005:** add desktop product UX, nearby-AP discovery, paged touchscreen target selection, TRACK / Change Target workflow, robust RFC1/RFD1/RFA1 control/reply handling, and physically qualify startup ordering plus bounded ECM recovery.
- **RFNAV-006 map feasibility spike:** prove that an OSM-derived offline regional vector package and Nano-X renderer are viable on physical Tomi, characterize LOD performance, memory, cell seams, coverage-edge behavior, relaunch stability and RFNAV/TomiDock coexistence.

Active production-design stage:

- **RFNAV-006:** integrate physically verified Tomi GPS state with RFN1 OBS history, Signal Rose/best-area guidance and the proven regional offline map architecture. First implementation work should verify the live GPS state-provider schema and jitter envelope, productionize the three LOD policies, then integrate geographic RF scoring without weakening RFNAV-005 operation when GPS is absent.

Later work may add geographic-session persistence, associated-uplink edge-case qualification, and BLE discovery/tracking. Those are not RFNAV-005 or current RFNAV-006 feasibility claims.

## Provenance

### RFNAV-002.2

Source/build evidence:

```text
/mnt/d/Codex/TT3/rfnav-0022-sys-evt-stack-20260926/
```

Qualified application SHA-256:

```text
35dbfa84259b2ca37e534154b43f86401df00dd8c2b759b59b487658ea818672
```

Full-stack qualification log identities:

```text
rfnav-0022-fullstack-udp.log
SHA-256 740d8acff4ae4bb3b8d884de3ab9eaa1ba614d2e73430f2faa3ed8ca33f060ee

rfnav-0022-fullstack-ping.log
SHA-256 63193cc218ea9b4f72a14188194afdd7911e71fb7a96c5ca3f5be146423ec7d9
```

### RFNAV-003

Primary source/build evidence:

```text
/mnt/d/Codex/TT3/rfnav-003-telemetry-20260926/
```

Qualified ESP application:

```text
SHA-256 30bb4799a0dfb6638a4582eb4c9f01f24f75b9e482c14a8c4147c120884600c4
```

Tomi receiver source and target binary:

```text
rfnav-rx.c
SHA-256 6ca6824603e61ca0b76f358b3e8d5c03947c51b66d12d72b33b924e5af74fccd

rfnav-rx
size 6512 bytes
SHA-256 f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6
```

Physical qualification log identities:

```text
rfnav-003-fullstack-udp.log
SHA-256 5a1145b8d9dc7313669beed94c8fdeba3466018df8760876f2513e4a9166f4f7

rfnav-003-fullstack-ping.log
SHA-256 dcd3b5fe3e3986a6b1f94b2aad2ea615fcb892d7cbce1fe2ef81d417864351cb

rfnav-v003-rx.log
SHA-256 c8137a7d1a121892f3e82d0eb35c568cfc0cb3bed4a9e0ce564f66bc7cb82292
```

### RFNAV-004

Primary source/build evidence:

```text
/mnt/d/Codex/TT3/rfnav-004-tomi-ui-20260926/
```

Corrected reviewed source identities:

```text
nxrfnav.c
SHA-256 5036c630b932bf915d955b78fe9639c942354ccad176f7fdc9092b691681da49

rfnav_model.c
SHA-256 c2ad0ddc42ae65483a8554f1ea8907e2485de98a576a4e392797f4e13401c0e0

rfnav_model.h
SHA-256 21e8f8297099f832c35c611641719d0b7b3de64f0911d724b456a44199656a7d

build-on-macbook.sh
SHA-256 a7d9ce987b268952c54c779dd754f45da3052698a35c5132744885d56ea3be23
```

Qualified target UI:

```text
nxrfnav-v001
size 13708 bytes
SHA-256 a7947016ae034199dcbab1240635f503cf2f003226c04e19ca0ccabda90ef365
```

Corrected RFNAV-004 evidence package supplied for independent review:

```text
SHA-256 104065668d53909f19b26b0baee52fb6f943b12631f85773a0e501213eaa6dba
```

The physical UI smoke test is supported by the operator-supplied live-screen photograph. The approximately 12-hour endurance result is operator-reported and is intentionally recorded as such rather than represented as a continuous instrumented log.

### RFNAV-005

Original RFNAV-005 implementation evidence:

```text
/mnt/d/Codex/TT3/rfnav-005-ux-target-selection-20260927/
```

Independent review and repaired-candidate evidence:

```text
/mnt/d/Codex/TT3/rfnav-005-astra-review-repair-20260927/
```

Astra review report:

```text
ASTRA_REVIEW_AND_REPAIR_REPORT.md
SHA-256 f70a85e6ed32f02fe773468b9d37b1a902a3ba8814b712733b93fa584592e8c2
```

Qualified ESP application:

```text
size 818848 bytes
SHA-256 1d8d1bce5e30f0f3ccc5ef8f0e03df1372bbe69729f1b46d8d35cfcadec83a09
```

MacBook target-build script used for the repaired Tomi UI:

```text
build-on-macbook.sh
SHA-256 514f6aff3bf3b109a03eeb3d59e54bb6b2bd09aef476eba8245fc5326d264af3
```

Qualified Tomi UI:

```text
nxrfnav-v002
size 20148 bytes
SHA-256 865cf76ddbe11ea1a383eea0ef107f23fbe85399d92933257317f28fbc680ba2
```

Operator-supplied instrumented physical-qualification capture used for final adjudication:

```text
size 52489 bytes
SHA-256 f964b6491cfa49c9eda954c30346765d1490bc41fb135506eebd9a197bf6f3f5
```

The preserved capture directly records healthy control traffic, successful manual discoveries, a real ECM unready interval, control-plane transition to `ready=0`, later `ready local=192.168.77.2:5516`, ECM/routing recovery in a new generation, and successful post-recovery manual discovery. Touch usability, two non-associated target selections, CLOSE/relaunch, duplicate-instance rejection, and UDP/5515 cleanup are operator-observed physical results.

### RFNAV-006 map feasibility

Primary feasibility/build evidence:

```text
/mnt/d/Codex/TT3/rfnav-map-feasibility-20260927/
```

The work unit contains the package-format definition, host generation/preview evidence, Tomi renderer source/build evidence, package results, commands and the physical test plan. Physical qualification results were then obtained on the real Tomi/TomiDock system using the frozen artifact identities recorded above. The current production conclusion is the adjudicated result of that host work plus the physical run: regional offline vector mapping is viable, while the feasibility 80 m/px LOD is intentionally rejected for production because it is both too slow and too visually dense.

Raw logs and experiment artifacts remain evidence rather than Git documentation. Retain them in the established Codex evidence warehouse. This canonical document records byte identities and adjudicated durable conclusions without reproducing private RF identifiers.