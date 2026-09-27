# RF Navigator (RFNAV)

## Purpose

RFNAV repurposes TT3 “Tomi” and TomiDock as a geographically aware RF signal navigator. Tomi provides GPS, touchscreen, Linux, and battery operation; the ESP32-S3 in TomiDock provides Wi-Fi/BLE radio capability. The intended product direction is to discover authorized nearby transmitters, select a target, measure signal strength repeatedly, and correlate that telemetry with Tomi-side state so Tomi can eventually help navigate toward a signal source.

This document is the canonical owner for RFNAV behavior, qualification state, protocol, and staged development. TomiDock USB/network architecture, lifecycle behavior, and electrical safety remain owned by [TomiDock Architecture](tomidock.md).

## Current qualified baseline

The current qualified RFNAV baseline is **RFNAV-003**.

It retains the RFNAV-002.2 discovery and focused-RSSI scan engine, the qualified `sys_evt` stack repair, and normal TomiDock CDC-ECM/routing behavior, and adds a Tomi-facing UDP telemetry path over the existing ECM link.

Qualified ESP application image:

```text
SHA-256: 30bb4799a0dfb6638a4582eb4c9f01f24f75b9e482c14a8c4147c120884600c4
```

ESP source candidate:

```text
/home/jazbob/opentom/lab-work/esp32-s3/tomi-s3-native-ecm-v002-routing-v001-nogpio-phase3-rfnav-v003
```

Qualified Tomi receiver:

```text
/mnt/sdcard/opentom/rfnav/bin/rfnav-rx
size: 6512 bytes
SHA-256: f55ee0c5c73a36df570d51ed8bf290f84b0547fc42204bb77d3442109f348ef6
```

The receiver was built with the pinned legacy Tomi toolchain, `arm-linux-gcc (GCC) 3.3.4`, in the Bookworm chroot. Its reviewed source SHA-256 is `6ca6824603e61ca0b76f358b3e8d5c03947c51b66d12d72b33b924e5af74fccd`.

## RFNAV-002 behavior retained

RFNAV-003 retains the RFNAV-002 behavior:

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

RFNAV-003 retains `CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE=4096` and the `SYS_EVT_HEALTH` diagnostic.

## RFNAV-003 Tomi telemetry architecture

RFNAV-003 adds a unidirectional telemetry path directly across the existing ECM subnet:

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

SSID is intentionally not carried in RFNAV-003. Target identity is the BSSID encoded as twelve hexadecimal digits without separators. Do not reproduce bench-specific BSSIDs in public canonical documentation.

### ESP execution model

Telemetry publication is subordinate to RFNAV scanning and TomiDock operation:

- RFNAV worker code copies bounded telemetry records into a 16-entry queue using zero wait;
- Wi-Fi / system-event callback context does not create sockets, format datagrams, or call `sendto()`;
- a separate priority-1 sender task with a 4096-byte stack owns formatting and nonblocking UDP transmission;
- destination is Tomi at `192.168.77.1:5515`;
- queue overflow, ECM-unready state, stale records, retry backoff, and send errors are counted rather than allowed to block or alter RFNAV/TomiDock control flow;
- telemetry queued while Tomi is absent is not allowed to grow without bound.

Low-frequency `RFNAV_TX_HEALTH` telemetry reports sender stack margin and counters alongside the existing `RFNAV_HEALTH` and `SYS_EVT_HEALTH` diagnostics.

### Tomi receiver

`rfnav-rx` is a small Tomi-native UDP receiver built with the pinned GCC 3.3.4 target toolchain. It binds UDP/5515, validates the `RFN1` prefix and bounded datagram length, records the sender IPv4 address, tracks receive count and sequence continuity, and reports gaps, rewinds, malformed records, and oversized records. It is a proof/diagnostic consumer, not the final RFNAV UI.

## RFNAV-003 physical qualification

RFNAV-003 was tested in the normal TomiDock topology with Tomi attached as USB host, CDC-ECM mounted, routing/NAPT and TCP/2323 active, the Tomi-native `rfnav-rx` receiver bound to UDP/5515, continuous LAN ping running, LAN-side UDP diagnostics active, and repeated RFNAV target scans continuing.

### Tomi receiver result

The Tomi receiver captured **484 consecutive RFN1 datagrams**:

```text
first captured sequence: 106
last captured sequence:  589
sender:                  192.168.77.2
receiver gaps:           0
receiver rewinds:        0
source changes:          0
malformed records:       0
oversized records:       0
```

All 484 captured records in this qualification window were `OBS` events for the selected target. The receiver began after RFNAV telemetry was already running, so absence of a `SELECT` record in this particular capture is not evidence that the protocol lacks or failed that event type.

The clean sequence span from 106 through 589 contains exactly 484 sequence values, independently matching the receiver's final `rx=484` count.

This physically proves direct ESP-to-Tomi RFN1 delivery over the ECM link from the ESP peer address rather than LAN-side delivery.

### ESP telemetry result

Before Tomi mounted, the sender correctly accumulated ECM-unready drops. At the last pre-mount health sample, `sent=0` and `ecm_drop` was increasing while ECM was not ready.

After the single observed USB MOUNT, TomiDock reported routing ready with ECM `192.168.77.2`, NAPT enabled, the TCP/2323 forward enabled, and no subsequent suspend/unmount in the qualification log.

During the qualified run:

- `RFNAV_TX_HEALTH` reached at least `sent=506`;
- `qdrop=0` throughout;
- `send_err=0` throughout;
- `stale_drop=0` throughout;
- `backoff_drop=0` throughout;
- `ecm_drop` stabilized at 96 after Tomi became available;
- sender-task stack high-water mark settled at 2104 free bytes;
- `sys_evt` high-water mark remained 1648 free bytes;
- RFNAV worker high-water mark remained 1148 free bytes;
- minimum-ever free heap reached 195732 bytes and then remained stable in the retained final health samples.

The `ecm_drop=96` count is expected pre-mount loss, not an in-service transport failure.

### RFNAV and TomiDock coexistence

The full ESP diagnostic observation covered approximately 19 minutes. It recorded:

- 538 target scan requests;
- 538 target scan starts;
- 538 target scan completions;
- all 538 scan completions with `status=0`;
- one isolated target miss, occurring before Tomi mounted;
- zero RFNAV rediscoveries;
- median target-scan duration about 123 ms;
- p95 target-scan duration about 124 ms;
- maximum target-scan duration 182 ms;
- 219 post-mount ECM snapshots, all linked, connected, mounted, unsuspended, and ready;
- 218 post-mount routing snapshots, all ECM-ready, Wi-Fi-ready, forwarding-enabled, NAPT-enabled, TCP/2323-enabled, and error-free;
- no panic, stack overflow, Guru Meditation, abort, reboot, Wi-Fi disconnect, USB suspend, or USB unmount.

The LAN ping capture recorded 1,828 successful replies across approximately 18 minutes 37 seconds with no failure/error lines. Observed ping statistics were approximately:

```text
median: 100 ms
mean:    99.3 ms
p95:    201 ms
p99:    267 ms
max:    451 ms
```

RFNAV-003 therefore passes physical qualification as the Tomi-facing telemetry transport baseline.

## Current design conclusions

- RFNAV discovery/focused scanning and TomiDock networking coexist stably.
- The RFNAV-002 reset root cause remains closed as insufficient `sys_evt` stack; RFNAV-003 preserves the qualified 4608-byte effective event-task allocation and measured 1648-byte worst-case free margin.
- RFNAV telemetry can be delivered directly from ESP `192.168.77.2` to Tomi `192.168.77.1` over CDC-ECM UDP without routing through the Wi-Fi LAN.
- The sender is isolated from `sys_evt`; telemetry transport failures are nonfatal and subordinate to RFNAV/TomiDock control flow.
- The 16-entry zero-wait queue and priority-1 sender task passed qualification with no queue drops or send errors while Tomi was present.
- The Tomi-native GCC 3.3.4 receiver accepted a lossless 484-record sequence window with no gaps, rewinds, source changes, malformed records, or oversized records.
- RFNAV-003 remains a telemetry transport milestone. `rfnav-rx` is diagnostic plumbing, not the final UI.

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
- **RFNAV-003:** establish and physically qualify direct Tomi-facing RFN1 telemetry over the existing ECM link, including a Tomi-native receiver.

Next planned stage:

- **RFNAV-004:** consume the qualified RFN1 stream on Tomi as a real user-facing live signal display. Keep the first 004 milestone intentionally small: current target identity/state, channel, RSSI, and obvious signal-strength trend/status. GPS correlation, directional navigation, mapping, persistence, BLE, and richer target-selection workflows remain later stages unless separately adjudicated into scope.

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

Reviewed source/build package supplied for independent review:

```text
rfnav-003-telemetry-20260926.tar
SHA-256 0b125f5cf0c85e59410ba6bb9d77d12dc5b17a53296d381aa2b2840152a8e33b
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

Physical qualification log identities supplied for adjudication:

```text
rfnav-003-fullstack-udp.log
SHA-256 5a1145b8d9dc7313669beed94c8fdeba3466018df8760876f2513e4a9166f4f7

rfnav-003-fullstack-ping.log
SHA-256 dcd3b5fe3e3986a6b1f94b2aad2ea615fcb892d7cbce1fe2ef81d417864351cb

rfnav-v003-rx.log
SHA-256 c8137a7d1a121892f3e82d0eb35c568cfc0cb3bed4a9e0ce564f66bc7cb82292
```

Raw logs remain evidence artifacts rather than Git documentation. Retain them in the established Codex evidence warehouse. This canonical document records byte identities and adjudicated durable conclusions without reproducing private RF identifiers.
