# RF Navigator (RFNAV)

## Purpose

RFNAV repurposes TT3 “Tomi” and TomiDock as a geographically aware RF signal navigator. Tomi provides GPS, touchscreen, Linux, and battery operation; the ESP32-S3 in TomiDock provides Wi-Fi/BLE radio capability. The intended product direction is to discover authorized nearby transmitters, select a target, measure signal strength repeatedly, and correlate that telemetry with Tomi-side state so Tomi can eventually help navigate toward a signal source.

This document is the canonical owner for RFNAV behavior, qualification state, protocol, UI behavior, and staged development. TomiDock USB/network architecture, lifecycle behavior, and electrical safety remain owned by [TomiDock Architecture](tomidock.md).

## Current qualified baseline

The current qualified RFNAV baseline is **RFNAV-004**.

RFNAV-004 retains the physically qualified RFNAV-003 ESP firmware and RFN1 transport unchanged and adds the first real Tomi-side Nano-X live signal instrument display.

Qualified ESP application image:

```text
SHA-256: 30bb4799a0dfb6638a4582eb4c9f01f24f75b9e482c14a8c4147c120884600c4
```

ESP source candidate:

```text
/home/jazbob/opentom/lab-work/esp32-s3/tomi-s3-native-ecm-v002-routing-v001-nogpio-phase3-rfnav-v003
```

Qualified Tomi UI:

```text
/mnt/sdcard/opentom/rfnav/ui/nxrfnav-v001
size: 13708 bytes
SHA-256: a7947016ae034199dcbab1240635f503cf2f003226c04e19ca0ccabda90ef365
```

Reviewed UI source:

```text
/home/jazbob/opentom/lab-work/tomi/nxrfnav-v001/nxrfnav.c
SHA-256: 5036c630b932bf915d955b78fe9639c942354ccad176f7fdc9092b691681da49
```

The UI was built with the pinned legacy Tomi toolchain, `arm-linux-gcc (GCC) 3.3.4`, in the Bookworm chroot and links only `libnano-X.so` and `libc.so.6` dynamically.

The earlier diagnostic receiver remains available at:

```text
/mnt/sdcard/opentom/rfnav/bin/rfnav-rx
size: 6512 bytes
SHA-256: f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6
```

`rfnav-rx` is diagnostic plumbing. `nxrfnav-v001` is now the qualified user-facing RFNAV consumer.

## RFNAV-002 behavior retained

RFNAV-004 retains the RFNAV-002 behavior through the unchanged RFNAV-003 ESP baseline:

1. one full active Wi-Fi discovery scan after network settlement;
2. deterministic target selection excluding the associated AP when possible;
3. preference for the strongest non-associated AP on a different channel, otherwise the strongest non-associated AP, otherwise the strongest visible AP;
4. repeated nonblocking scans filtered to the selected BSSID and channel;
5. a nominal 2-second quiet interval between target scans;
6. target RSSI reporting when observed;
7. rediscovery after five consecutive valid target misses;
8. target invalidation and fresh discovery when Wi-Fi state changes.

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

RFNAV-003 and RFNAV-004 retain `CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE=4096` and the `SYS_EVT_HEALTH` diagnostic.

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

This extended run materially exceeds the original short smoke/soak intent and qualifies `nxrfnav-v001` as the current live RFNAV user interface for the observed normal target-tracking workload.

Host-side qualification separately covers bounded RFN1 parsing, all five protocol event types, malformed/oversized input, sequence continuity/wrap/gap/rewind handling, source and epoch changes, stale/no-telemetry timeouts, trend behavior, and gauge clamping. Those host tests complement but do not imply that every exceptional RFN1 state was deliberately forced during the 12-hour physical run.

## Current design conclusions

- RFNAV discovery/focused scanning and TomiDock networking coexist stably.
- The RFNAV-002 reset root cause remains closed as insufficient `sys_evt` stack; the qualified RFNAV-003 ESP baseline remains unchanged beneath RFNAV-004.
- RFNAV telemetry is delivered directly from ESP `192.168.77.2` to Tomi `192.168.77.1` over CDC-ECM UDP without routing the product path through the Wi-Fi LAN.
- The sender remains isolated from `sys_evt`; telemetry transport failures are nonfatal and subordinate to RFNAV/TomiDock control flow.
- RFNAV-003 proved the transport losslessly over a 484-record receiver window.
- RFNAV-004 proves that Tomi can consume that stream directly as a practical live Nano-X field display.
- `nxrfnav-v001` is the current qualified user-facing RFNAV application.
- RSSI is treated as received signal strength, not physical distance.
- The approximately 12-hour RFNAV-004 endurance result is strong operational evidence but remains an operator observation rather than an instrumented continuous log.

## Safety and operating constraints

RFNAV inherits all TomiDock electrical and USB safety constraints. Never connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. Use the isolation sequence documented in [TomiDock Architecture](tomidock.md).

Do not reintroduce GPIO4 monitoring or gating. GPIO4 is unconnected and has no runtime role in the current TomiDock design.

Do not perform routine USB detach-anomaly cycling as an RFNAV qualification step. Preserve an unsolicited lifecycle anomaly if one occurs during otherwise authorized work, but RFNAV development does not reopen that retired investigation.

## Development stages

Completed:

- **RFNAV-001:** prove full background Wi-Fi discovery scanning can coexist with TomiDock networking.
- **RFNAV-002:** prove discovery -> deterministic target selection -> repeated focused RSSI sampling while TomiDock remains operational.
- **RFNAV-002.1:** instrument and identify the unsolicited reset as `sys_evt` stack overflow.
- **RFNAV-002.2:** increase and instrument `sys_evt` stack; pass standalone and full-stack physical qualification.
- **RFNAV-003:** establish and physically qualify direct Tomi-facing RFN1 telemetry over the existing ECM link, including a Tomi-native diagnostic receiver.
- **RFNAV-004:** build and physically qualify the first Tomi-native live RF signal display, including extended approximately 12-hour continuous operation.

Next planned stage:

- **RFNAV-005:** scope is not yet adjudicated. The leading product directions are richer operator target-selection workflow and GPS correlation/navigation context. Choose the next milestone explicitly before implementation rather than combining both by default.

Later work may also add mapping, persistence, and BLE discovery/tracking. Those are not RFNAV-004 claims.

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

Raw logs and experiment artifacts remain evidence rather than Git documentation. Retain them in the established Codex evidence warehouse. This canonical document records byte identities and adjudicated durable conclusions without reproducing private RF identifiers.
