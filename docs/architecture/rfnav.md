# RF Navigator (RFNAV)

## Purpose

RFNAV repurposes TT3 “Tomi” and TomiDock as a geographically aware RF signal navigator. Tomi provides GPS, touchscreen, Linux, and battery operation; the ESP32-S3 in TomiDock provides Wi-Fi/BLE radio capability. The intended product direction is to discover authorized nearby transmitters, select a target, measure signal strength repeatedly, and eventually correlate that telemetry with Tomi GPS and UI state so Tomi can help navigate toward a signal source.

This document is the canonical owner for RFNAV behavior, qualification state, and staged development. TomiDock USB/network architecture, lifecycle behavior, and electrical safety remain owned by [TomiDock Architecture](tomidock.md).

## Current qualified baseline

The current qualified RFNAV baseline is **RFNAV-002.2**. It adds focused Wi-Fi target tracking to the existing TomiDock ESP32-S3 firmware while preserving normal CDC-ECM networking, routing/NAPT, and TCP/2323 forwarding.

Qualified application image:

```text
SHA-256: 35dbfa84259b2ca37e534154b43f86401df00dd8c2b759b59b487658ea818672
```

Source candidate:

```text
/home/jazbob/opentom/lab-work/esp32-s3/tomi-s3-native-ecm-v002-routing-v001-nogpio-phase3-rfnav-v0022
```

RFNAV-002.2 was deployed application-only. The bootloader and partition table were not intentionally changed for qualification.

## RFNAV-002 behavior

RFNAV-002 performs:

1. one full active Wi-Fi discovery scan after network settlement;
2. deterministic target selection excluding the associated AP when possible;
3. preference for the strongest non-associated AP on a different channel, otherwise the strongest non-associated AP, otherwise the strongest visible AP;
4. repeated nonblocking scans filtered to the selected BSSID and channel;
5. a nominal 2-second quiet interval between target scans;
6. target RSSI reporting when observed;
7. rediscovery after five consecutive valid target misses;
8. target invalidation and fresh discovery when Wi-Fi state changes.

Target scans retain the RFNAV-001 scan parameters: active maximum dwell 120 ms and home-channel dwell 30 ms. Wi-Fi power save remains the ESP-IDF 5.5.5 default `WIFI_PS_MIN_MODEM`. RFNAV does not use GPIO4 and does not alter TomiDock USB lifecycle policy.

## RFNAV-002.1 panic diagnosis

The initial RFNAV-002 implementation functioned correctly but was not stable: an unsolicited ESP restart occurred during physical qualification. RFNAV-002.1 added reset-reason, RFNAV-task stack, and heap telemetry without changing RFNAV scan behavior.

A standalone UART experiment with Tomi completely disconnected reproduced the failure three times. Each failure reported:

```text
***ERROR*** A stack overflow in task sys_evt has been detected.
```

The reproduced failures occurred at approximately 332.5 s, 204.8 s, and 192.1 s of uptime. Each followed `TARGET_SCAN_REQUEST` and occurred before `TARGET_SCAN_STARTED`. This establishes `sys_evt` stack exhaustion under the repeated target-scan workload; it does not establish that `esp_wifi_scan_start()` itself is defective.

For this ESP-IDF 5.5.5 build:

```text
CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE = 2304
TASK_EXTRA_STACK_SIZE                   = 512
old effective sys_evt stack             = 2816 bytes
```

`CONFIG_LWIP_TCPIP_CORE_LOCKING` and `CONFIG_LIBC_NEWLIB_NANO_FORMAT` were both disabled.

## RFNAV-002.2 repair

RFNAV-002.2 changes the configured system-event task stack to 4096 bytes and adds low-frequency `sys_evt` high-water telemetry. The effective event-task stack therefore becomes:

```text
4096 + 512 = 4608 bytes
```

No intentional RFNAV scan, target-selection, Wi-Fi, USB, ECM, routing, NAPT, TCP/2323, GPIO4, or worker scheduling behavior changed from RFNAV-002.1.

Physical instrumentation measured a worst-case `sys_evt` high-water mark of 1648 free bytes. Therefore the maximum observed event-task stack use is approximately:

```text
4608 total - 1648 free = 2960 bytes used
```

The prior effective allocation was only 2816 bytes, so the measured workload exceeded the old allocation by approximately 144 bytes. The 4608-byte allocation leaves approximately 35.8% of the event-task stack free at the observed high-water mark.

This closes the RFNAV-002.1 reset root cause as `sys_evt` stack exhaustion under the RFNAV targeted-scan workload.

## Physical qualification

### Standalone qualification

RFNAV-002.2 was first qualified with Tomi disconnected and the ESP operating without the Tomi CDC-ECM host path. It ran beyond the previous 192–332 s failure window and then through an extended instrumented run without panic or reset. `SYS_EVT_HEALTH` remained at 1648 free bytes; the RFNAV worker high-water mark remained about 1180 free bytes; heap telemetry was stable.

The ESP reached roughly 49 minutes of uptime without another panic during the standalone qualification sequence.

### Full TomiDock qualification

RFNAV-002.2 was then tested in the normal TomiDock topology with Tomi attached as USB host, CDC-ECM mounted, routing/NAPT active, TCP/2323 exercised, continuous LAN ping running, UDP diagnostics active, and repeated RFNAV target scans.

The full-stack observation covered approximately 24 minutes. After the TCP/2323 path was accepted, the system remained qualified for more than 22 additional minutes.

Observed full-stack results:

- 674 target scan requests;
- 674 target scan starts;
- 674 target scan completions;
- zero nonzero scan-status results;
- eight target misses, all isolated `consecutive=1` events;
- zero RFNAV rediscoveries;
- median target-scan duration about 123 ms;
- maximum target-scan duration 184 ms;
- 2,269 continuous ping samples with zero failures/timeouts;
- ping median about 100 ms, p95 about 200 ms, maximum 455 ms;
- all observed ECM snapshots remained linked, connected, mounted, unsuspended, and ready;
- all observed routing snapshots remained ECM-ready, Wi-Fi-ready, forwarding-enabled, NAPT-enabled, and TCP/2323-enabled;
- TCP acceptance advanced to `tcp_accepts=1` and remained there;
- RFNAV worker stack high-water mark remained 1180 free bytes;
- `sys_evt` high-water mark reached 1648 free bytes and did not decrease further;
- minimum-ever free heap reached 201020 bytes and then remained stable;
- no panic, stack overflow, Guru Meditation, abort, reboot, Wi-Fi disconnect, USB suspend/unmount, or RFNAV rediscovery was observed.

RFNAV-002.2 therefore passes both standalone and full TomiDock physical qualification.

## Current design conclusions

- RFNAV background/focused Wi-Fi scanning can coexist with normal TomiDock networking.
- The RFNAV-002.1 unsolicited resets were caused by insufficient ESP-IDF `sys_evt` stack, not by a requirement for Tomi/CDC-ECM participation.
- A 4096-byte configured event-task stack, 4608 effective bytes in this build, is physically qualified for the current workload with 1648 bytes of measured worst-case free margin.
- The RFNAV worker's own 4096-byte allocation is not the source of the historical panic; its measured high-water margin remains healthy.
- RFNAV target misses observed during qualification were transient and did not approach the five-miss rediscovery threshold.
- The current qualified RFNAV baseline remains `WIFI_PS_MIN_MODEM`; the separate built-only `WIFI_PS_NONE` latency candidate in TomiDock documentation is not part of RFNAV-002.2.
- RFNAV-002.2 should remain the working baseline for subsequent stages rather than creating a cleanup-only firmware revision.

## Safety and operating constraints

RFNAV inherits all TomiDock electrical and USB safety constraints. In particular, never connect ESP debug/programming USB to DEAN-PC while Tomi remains attached to the powered TomiDock path. Use the isolation sequence documented in [TomiDock Architecture](tomidock.md).

Do not reintroduce GPIO4 monitoring or gating as part of RFNAV work. GPIO4 is unconnected and has no runtime role in the current TomiDock design.

Do not perform routine USB detach-anomaly cycling as an RFNAV qualification step. Preserve an unsolicited lifecycle anomaly if one occurs during otherwise authorized work, but RFNAV development does not reopen that retired investigation.

## Development stages

Completed:

- **RFNAV-001:** prove full background Wi-Fi discovery scanning can coexist with TomiDock networking.
- **RFNAV-002:** prove discovery -> deterministic target selection -> repeated focused RSSI sampling while TomiDock remains operational.
- **RFNAV-002.1:** instrument and identify the unsolicited reset as `sys_evt` stack overflow.
- **RFNAV-002.2:** increase and instrument `sys_evt` stack; pass standalone and full-stack physical qualification.

Next planned stage:

- **RFNAV-003:** establish a stable Tomi-facing telemetry path for RFNAV target observations over the existing ECM link. The first 003 milestone should transport target identity/state, channel, RSSI, observation/miss state, and generation/sequence information to Tomi without adding GPS correlation, Nano-X UI, BLE, mapping, or navigation logic yet.

Later stages may add Tomi UI, GPS correlation, directional/navigation logic, persistence, and BLE discovery/tracking. Those are not current RFNAV-002 claims.

## Provenance

Primary source/build evidence:

```text
/mnt/d/Codex/TT3/rfnav-0022-sys-evt-stack-20260926/
```

RFNAV-002.2 application SHA-256:

```text
35dbfa84259b2ca37e534154b43f86401df00dd8c2b759b59b487658ea818672
```

Preserved source/build package supplied for independent review:

```text
rfnav-0022-sys-evt-stack-20260926.tar
SHA-256 cd54d999ba2caf96348db7c0ea0041de3fcd1b054b3dc5533394524d50f23284
```

Standalone qualification log identities:

```text
rfnav-0022-standalone-udp.log
SHA-256 70df6fdd1ad80921a565e302541034a85ac0071e9146fde25724974f649c8214

serial-monitor(2).log
SHA-256 d302d1f0f66014be7c1382c8a75e1a78fbfe8daeef5356d58101619b5c5f9122
```

Full-stack qualification log identities:

```text
rfnav-0022-fullstack-udp.log
SHA-256 740d8acff4ae4bb3b8d884de3ab9eaa1ba614d2e73430f2faa3ed8ca33f060ee

rfnav-0022-fullstack-ping.log
SHA-256 63193cc218ea9b4f72a14188194afdd7911e71fb7a96c5ca3f5be146423ec7d9
```

The raw logs remain evidence artifacts rather than Git documentation. They should be retained in the established Codex evidence warehouse; this document records their byte identities and adjudicated durable conclusions without reproducing private RF identifiers.
