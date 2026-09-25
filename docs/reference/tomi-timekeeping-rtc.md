# Tomi RTC and Timekeeping Reference

## Scope and evidence model

This page records what is established about TT3 “Tomi” real-time clock, Linux
wall time, GPS-derived time/date, synchronization, boot-time initialization,
and local-time presentation. It owns relationships among these domains; it is
not a GPS protocol or graphical-display reference.

Evidence labels:

- **LIVE OBSERVED** — present in a dated Tomi terminal transcript or log.
- **EXACT BINARY** — static result tied to a hash-identified executable.
- **SOURCE-CORRELATED** — defined by inspected OpenTom source/configuration;
  the exact source-to-running-kernel match is not cryptographically proved.
- **COMPARATIVE ONLY** — evidence from a different TomTom unit/build.
- **UNRESOLVED** — the preserved material does not decide the question.

Keep these concepts distinct: receiver/navigation time; GLL/`glgps`
interpretation; NMEA RMC UTC date/time; Linux `CLOCK_REALTIME`; monotonic
elapsed time/uptime; hardware RTC; filesystem timestamps; and the time an
application presents to a user after any timezone conversion.

## Surviving time-flow model

```text
GPS/GLL-derived UTC in ttgpsd state
    ──[LIVE OBSERVED custom script]──> Linux wall clock (`date -u -s`)
    ──[LIVE OBSERVED command returned success]──> RTC (`hwclock -w -u`)

RTC ──[SOURCE-CORRELATED kernel option/path]──> Linux wall clock at RTC probe

RTC ──[EXACT BINARY only in later 2018 GLL specimen; live execution unknown]──>
     GLL precise-time assistance

nxdclock displayed time source / timezone conversion: UNKNOWN
```

The arrows have different evidence strengths and contexts. In particular, the
later GLL RTC-provider strings are not proof that this provider ran on Tomi.
The older hash-identified Tomi GLL binary does not contain that provider.

## Hardware RTC

**LIVE OBSERVED:** Tomi kernel-log snapshots identify Samsung S3C2412 and show
the `S3C2410 RTC` driver registering; the driver reports that the RTC was
disabled and re-enables it. This supports an S3C24xx on-SoC RTC interface in
the observed kernel, rather than evidence for a separate RTC chip. The exact
board-level RTC power/backup arrangement is not established.

**SOURCE-CORRELATED:** the inspected OpenTom `drivers/char/s3c2410-rtc.c`
implements the internal RTC, standard `RTC_RD_TIME` and `RTC_SET_TIME` ioctls,
and alarm operations. With `CONFIG_S3C2410_RTC_SETTIMEOFDAY`, its RTC probe
reads the RTC and calls `do_settimeofday()` to initialize Linux wall time.
The inspected generated config enables that option. The source tree is not
proved byte-identical to the kernel in the dated runtime capture, so this is
not a definitive trace of every live boot.

In this same inspected configuration, `CONFIG_S3C2410_RTC_GETTIMEOFDAY` is
undefined. The source therefore does not select its optional system-time-to-
RTC writeback callback on shutdown or suspend. With
`CONFIG_S3C2410_RTC_SETTIMEOFDAY` enabled, the inspected PM resume path reads
the RTC and resets Linux wall time. These are source/configuration findings,
not a Tomi suspend/resume measurement or proof that the exact live kernel used
these options.

The RTC is written in the custom Aug. 24 GPS run via BusyBox `hwclock -w -u`;
the script logs success only after that command exits successfully. No
syscall/ioctl trace or independent before/after RTC read was captured. There
is no Tomi test proving time retention across complete power loss, RTC backup
source, alarm use, or long-term accuracy. A successful write command is not
proof of those properties.

The historical 2009 `glgps` executable lacks the `GlRtc*` adapter, `/dev/rtc0`
path, `glrtc-rtc-file` key, and the donor's private RTC ioctl constants. A
later 2018 GLL specimen contains those strings and provider code, but string
presence and a similar file size do not prove live RTC access. The August
2026 running `glgps` was not hashed on Tomi.

## Linux system time and boot-time initialization

The live custom GPS script explicitly invokes `/bin/busybox date -u -s` with
the validated GPS-derived timestamp. It rejects absent state, invalid
`time_valid`/`date_valid`, empty ISO text, and years before 2020 or after 2036.
The script then runs `/bin/busybox hwclock -w -u`. It retries unsuccessful
attempts every 60 seconds and, after success, intends hourly updates. These
are findings about that preserved custom run, not proof that the stock
TomTom application uses the same policy.

The Aug. 24 transcript records nine successful `GPS -> system -> RTC` log
entries in the same second (04:05:11Z) as the later `glgps` FIN,1 shutdown.
This is an abnormal burst and a temporal correlation only. It does not prove
that the script caused the shutdown or that the RTC caused the later GPS date.
The shutdown investigation found `glgps`'s own ASIC-death classification and
left the lower-level trigger unresolved. See [GPS/glgps reference](tomi-gps-glgps.md).

The live kernel log identifies the RTC driver during boot. The inspected
OpenTom RTC source can seed `CLOCK_REALTIME` from RTC at probe when the
build-time option is enabled, but the exact running-kernel configuration and
the first wall-clock value before userspace are not directly captured. No
Tomi-specific bootloader RTC read/display or pre-Linux time value has been
established. A bounded review of the 5.5279 bootloader reference found no
RTC/time behavior documented there; this is not an exhaustive negative over
every bootloader instruction. The inspected bootloader package is static
update-package evidence, not a live NOR execution trace; do not infer its RTC
behavior.

## GPS time and the 2007 / 2026 date findings

The GPS reference owns receiver/GLL protocol details. For timekeeping,
**LIVE OBSERVED** evidence from the custom Aug. 24, 2026 run includes a valid
RMC date `240826` and `ttgpsd` state `utc_iso8601=2026-08-24T04:05:11.000Z`.
The transcript reports GLL RID 2.16.201. No raw UART capture or on-device
hash binds the running executable to a preserved binary.

The older captured Tomi `glgps` is **EXACT BINARY**: 1,090,108 bytes,
SHA-256 `c807db452a5a84b48988e30034910a201e73cf2c86ff9252287782e919dcd4b7`,
GLL 2.15.0 build 65947 (2009). In the historical controlled epoch test,
Linux wall-time manipulation and a cold-start job did not prevent a valid GPS
fix from yielding a date in 2007. The epoch report's dated comparison
(2007-01-03 versus 2026-08-19) shares a modulo-1024 GPS week and differs by
1024 weeks (7168 days). The separate Aug. 24, 2026 RMC is a later current-era
observation, not the date used for that exact arithmetic comparison. The
analysis attributes the old result most strongly to unassisted old
GLL/Barracuda epoch unfolding plus absence of the full-epoch RTC assistance
adapter in that exact binary. The precise point of epoch selection (ARM GLL
versus receiver state/firmware) remains unresolved.

The later preserved GLL executable is 1,176,168 bytes, SHA-256
`45fcfa0ae50fb915ba536feda712769324c2ea25ab16acc01f3d0ce9048dd099`, GLL
2.16.201 build 354431 (2018). It includes RTC-provider strings. The later
2026 RMC observation is therefore not a contradiction of the scoped 2007
test: the builds and runtime contexts differ, while the live 2026 executable
identity and the cause of its current-era date remain unproved. Candidate
factors include build/provider differences, initial RTC/system state,
receiver state, configuration, and startup ordering; none is selected as the
cause without a hash-bound trace.

## Monotonic time versus wall time

The shutdown analysis identifies `times()`-based kernel ticks for the
hash-identified 2018 `glgps` watchdog/timer logic; `gettimeofday()` is used
for wall-time retrieval there. The live transcript also records `/proc/uptime`
and GPS uptime counters. These are elapsed-time observations, not RTC values
or local calendar time. The evidence does not establish that every application
timer, logger timestamp, or sleep uses the same clock source. No measured
monotonic discontinuity across a wall-clock adjustment is recorded.

## `nxdclock`, timezone, and filesystem presentation

**LIVE OBSERVED:** a Tomi process snapshot shows
`/mnt/sdcard/opentom/thomas/bin/nxdclock` running with Nano-X. The executable
and its source were not found in the preserved TT3 evidence searched for this
migration. The unrelated OpenTom `applications/src/tools/nxclock.c` calls
`gettimeofday()`/`localtime()`, but is an analog demo and is not identified as
the deployed `nxdclock`. Therefore `nxdclock`'s clock API, refresh interval,
RTC access, UTC/local conversion, and timezone handling are unknown. See the
[display reference](tomi-display-nanox.md) for graphics/process facts only.

The custom synchronizer explicitly uses UTC (`date -u`, UTC ISO timestamps),
and RMC is UTC. This does not establish the Tomi-wide timezone policy or what
the UI displays. No supported evidence of a `TZ` setting, `/etc/localtime`
selection, daylight-saving policy, or user-selected offset was found in the
time-related evidence. Filesystem modification dates (including old-looking
GPS data timestamps) are not treated as RTC readings.

The [runtime ABI reference](tomi-runtime-abi.md) records BusyBox `hwclock`
availability; that establishes an applet, not correct RTC operation or stock
startup use.

## Comparative findings

- **COMPARATIVE ONLY / same software stack:** a separate 2010 TT1 GLL 2.16.201
  donor contains `GlRtc*` and `/dev/rtc0` provider logic with private ioctls.
  It is not a Tomi executable. A TT1 RTC0-alias/probe run is also not Tomi
  evidence.
- **NON-TRANSFERABLE:** do not transfer RTC addresses, device paths, driver
  contracts, or boot behavior from TT1, TJ/Atlas III, or TT4 to Tomi.
- Tomi-specific S3C2412 and observed RTC-driver facts are summarized in the
  [device profile](../devices/tt3-tomi.md). Boot-stage boundaries are in the
  [boot-chain reference](../architecture/tomi-boot-chain.md).

## Negative findings and known unknowns

- No direct Tomi readback of RTC value or syscall-level proof of RTC ioctls in
  the custom GPS run; the successful BusyBox command is the available result.
- No demonstrated RTC retention across total power loss, backup-cell/source
  identity, accuracy, or alarm behavior.
- No hash of the Aug. 24 live `glgps`, no raw GPS UART capture, and no trace
  showing whether that live process opened an RTC device or consumed full-week
  assistance.
- No proof that the `nxdclock` application reads Linux time, RTC directly, or
  converts UTC to local time.
- No established Tomi timezone/DST policy, exact first Linux wall-clock value,
  or bootloader-visible clock behavior.
- The nine rapid sync successes and `FIN,1` are correlated in time, not
  causally linked.
- The 2007 result remains valid for its hash-identified 2009 binary/test; the
  later 2026 date remains a separately scoped live observation. The cause of
  the difference is unresolved.

## Canonical references and provenance

- [Tomi GPS/glgps](tomi-gps-glgps.md) — GPS runtime, binary identities, and
  epoch investigation.
- [Tomi display/Nano-X](tomi-display-nanox.md) — `nxdclock` process/graphics
  scope.
- [Tomi runtime ABI](tomi-runtime-abi.md) — BusyBox command availability.
- [Tomi device profile](../devices/tt3-tomi.md) and
  [Tomi boot chain](../architecture/tomi-boot-chain.md) — hardware and boot
  evidence boundaries.
- [Evidence Index](../../evidence/INDEX.md) — selected durable evidence
  anchors.

The source/evidence inventory and reconciliation are preserved outside LUCE
at `/mnt/d/Codex/TT3/luce-bootstrap-tomi-timekeeping-rtc-reconciliation-2026-09-24.md`.
The external work units remain authoritative for raw captures; no source,
firmware, device, or Codex index was changed.
