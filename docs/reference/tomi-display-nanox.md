# Tomi Display, Framebuffer, and Nano-X Reference

## Scope and evidence rules

This reference describes the display software path and graphical programming model evidenced for TT3 “Tomi.” It does not identify the physical LCD, define battery semantics, or document general shell compatibility. See [the Tomi device profile](../devices/tt3-tomi.md), [power/battery reference](tomi-power-battery.md), [runtime ABI](tomi-runtime-abi.md), and [lab environment](../environment/lab-environment.md) for those topics.

Evidence labels are used narrowly:

- **LIVE OBSERVED** — captured on-device process, command, or visible behavior.
- **SOURCE** — present in the inspected OpenTom/Nano-X source or configuration; not proof that the deployed image used that exact revision.
- **APPLICATION ASSUMPTION** — a candidate's layout or code constant, not independent display measurement.
- **UNKNOWN** — not established by the reviewed material.

The WSL OpenTom tree was not pristine and is not identity-bound to the installed Tomi binaries. Its Nano-X configuration/source is therefore comparative source evidence, not an exact deployed-build claim.

## Display-stack model

```text
Tomi Linux framebuffer (/dev/fb; /dev/fb0 is made as a symlink by startup)
                 ↑
       Microwindows / Nano-X server (nano-X)
          ↕ local client/server IPC
  nanowm + Nano-X clients (nxdclock, nxbg, utilities)
```

**LIVE OBSERVED:** A Tomi telnet capture shows distinct `nano-X`, `nanowm`, and `nxdclock` processes running together (PIDs 337, 338, and 356 in that captured session). Its startup-script excerpt exports `FRAMEBUFFER=/dev/fb`, sets `TSLIB_FBDEVICE=/dev/fb`, creates `/dev/fb0` as a symlink to `/dev/fb`, and derives `NANOX_YRES` from `fbset -s`. This establishes the configured access path and separate process model for that session; the captured text does not include `fbset`'s returned geometry or framebuffer ioctl values.

**SOURCE:** The inspected Microwindows configuration selects `ARCH=LINUX-ARM`, `NANOX=Y`, shared libraries, `LINK_APP_INTO_SERVER=N`, and shared-memory protocol support. Nano-X client `GrOpen()` opens an AF_UNIX stream socket by default, with `NXDISPLAY` or `GR_NAMED_SOCKET` selecting its pathname. That supports a separate server/client architecture in this source configuration. Exact socket pathname and shared-memory use by the installed Tomi build have not been independently captured. The live process list corroborates a separate server, not the particular IPC implementation.

## Linux framebuffer

| Property | Finding | Evidence class |
|---|---|---|
| Device node | `/dev/fb`; startup creates `/dev/fb0` as its symlink | LIVE OBSERVED startup text |
| Geometry | Startup queries `fbset -s`; no result captured. `480x272` appears in application reports/layouts | UNKNOWN as measured mode; APPLICATION ASSUMPTION in those layouts |
| Virtual geometry, bpp, channel order, pixel packing, stride, memory size | No Tomi live ioctl or `fbset` output found in reviewed evidence | UNKNOWN |
| Physical/virtual framebuffer address | No Tomi-specific live mapping evidence established here | UNKNOWN |
| Ioctl/mmap/direct writes | No Tomi direct-framebuffer experiment or measured write/mmap result found in the indexed and bounded-reviewed artifacts | UNKNOWN; no claim that such work never occurred |
| Rotation, blanking, palette, page flipping | Not established | UNKNOWN |

The inspected OpenTom Microwindows config contains `SCREEN_PIXTYPE=MWPF_TRUECOLOR565`; this is a **SOURCE configuration value only**. It is not sufficient to state that the active Tomi framebuffer is RGB565. Likewise, `480x272` in `nxbg`/nxbattery work is a coordinate/layout basis, not a recovered kernel mode or panel identity. Do not use graphical output to resolve the display-model discrepancy recorded in the device profile.

## Nano-X / Microwindows runtime

### Libraries, client API, and rendering

The preserved nxbattery V001/V002 ARM ELF reports `libnano-X.so` as a dynamic dependency, alongside libc. V001 is 12,620 bytes (SHA-256 `211578efb932cca7920971f3eebaf2ed0520abd87ad6caded6f4f51e779eba5e`); V002 is 12,716 bytes (SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`). These identify preserved candidate artifacts, not the binary or Nano-X library currently installed on Tomi. Candidate instructions use `/mnt/sdcard/opentom/lib` for shared-library resolution; installed-library hash/version remains unknown.

Representative tested source patterns use `GrOpen()`, `GrNewWindowEx()` with `GR_ROOT_WINDOW_ID` parent, `GrNewGC()`, `GrSetGC*()`, `GrCreateFontEx()`, `GrSelectEvents()`, `GrMapWindow()`, `GrText()`, `GrFillRect()`, `GrFlush()`, and explicit resource cleanup. `GrGetNextEventTimeout()` drives periodic updates and receives exposure/close-request events. These demonstrate client construction and candidate-host testing; they do not alone prove target rendering fidelity.

### Fonts and input

nxbattery requests `GR_FONT_SYSTEM_VAR` at 18×18 in its source. That is the application's font request, not a verified physical font file/name or measured glyph metric on Tomi. The inspected Microwindows config enables native FNT/FNTGZ and PCF-GZ support, while FreeType, PCF (uncompressed), and Type 1 support are disabled. Those are source-config observations only; which font assets are installed/loaded by the deployed server and whether fallback occurred are unknown. No Tomi-specific anti-aliasing or font-rendering measurement is preserved.

Startup exports tslib paths (`/dev/input/event0`, `/dev/fb`, a `ts.conf`, plugin directory, and calibration file), and first-light package material contains tslib components. This is evidence of configured input plumbing, not proof of live touch events, calibration quality, Nano-X event delivery, or current production use. The [touchscreen/input reference](tomi-touchscreen-input.md) owns acquisition and calibration evidence.

## Rendering, windows, and background behavior

- The run-ready status-disc cue, the separate whole-root-background mechanism, and their candidate-validation limits are described together under [Diagnostic cues](#diagnostic-cues) below.
- No modern alpha-compositing guarantee is established. Background repaint, expose behavior, persistence under the active desktop, and z-order/coexistence must be treated as target-specific until directly observed.
- V001/V002 nxbattery candidate windows are borderless, non-focus, non-resizable children of the Nano-X root. Source places V001 at default `(195,121)` with `275×68`; V002 retains width/coordinates and changes height to 90, drawing its added line at `(10,65)`. V002 report explicitly leaves target font rendering and coexistence untested. These are application geometry, not global window-manager or screen bounds.

## Graphical application cases

### Clock — both graphics and a separate timekeeping concern

The Tomi telnet capture establishes that `/mnt/sdcard/opentom/thomas/bin/nxdclock` was running beside `nano-X` and `nanowm`. Existing nxbattery documentation gives nxdclock v0.5 layout values, but its own candidate was not bench-validated in that nxbattery work unit. Thus the clock is a **GRAPHICAL_APPLICATION** here only for process coexistence and its reported layout. The clock's displayed time source, RTC synchronization, timezone and boot-time behavior are owned by the [timekeeping/RTC reference](tomi-timekeeping-rtc.md). See also [GPS/glgps findings](tomi-gps-glgps.md).

The unrelated OpenTom `applications/src/tools/nxclock.c` is an analog Nano-X demo using `gettimeofday()`/`localtime()` and drawing primitives. It is source material, not identified as Tomi's deployed `nxdclock`.

### nxbattery — graphical facts only

V001 and V002 provide source/build validation for dynamic Nano-X clients, root-child window setup, text and bar drawing, a system-variable font request, exposure repaint, timer-driven update, and cleanup. V002's extra raw status line is source-positioned after gauge drawing. Host tests and ARM linking do not establish target font spacing, final pixel placement, visual coexistence, repaint appearance, or the installed Nano-X ABI. Battery model and signal meaning remain in [the power/battery reference](tomi-power-battery.md).

### Diagnostic cues

The established `/mnt/sdcard/opentom/nxbg` command paints a status disc on
`GR_ROOT_WINDOW_ID`: a 28-pixel white outer disc and 24-pixel inner disc at
application coordinates `(456,248)`, based on a 480×272 layout assumption.
`green` is literal `GR_RGB(0,255,0)`; historical `black` is a compatibility
alias for the normal gray `GR_RGB(96,96,96)`, not literal black. This is a
root-window drawing cue, not a whole-screen background. The original
run-ready candidate reused that established cue, but it was unsuitable for
the requested whole-desktop BLACK/GREEN presentation. The reported smoke
showed it did not provide a whole-desktop ready indication; it did not show
that the status disc itself was broken or invisible.

The correction adds a separate candidate helper, `tomi-rootbg black|green`.
It zero-initializes `GR_WM_PROPERTIES`, sets `flags` to
`GR_WM_FLAGS_BACKGROUND` and `background` to the requested color, then calls
`GrSetWMProperties(GR_ROOT_WINDOW_ID, &props)`, flushes, and exits. The
requests are literal BLACK `GR_RGB(0,0,0)` and GREEN `GR_RGB(0,255,0)`. OpenTom
Nano-X source shows that changing the realized unbuffered root background
causes full-window expose/repaint processing; this source path is not proof
that the candidate visibly changed the live Tomi desktop. A root-window
status disc and a whole-root background are different Nano-X mechanisms.

The `tomi-runready` application contract is to start BLACK and sample valid
battery and hardware-status reads, asserted external power, and battery voltage
of at least 4190 mV approximately every 10 seconds. The threshold was selected
for run #4 qualification; it is not SOC, full charge, charger completion,
capacity, or a runtime guarantee (see [power and battery semantics](tomi-power-battery.md)).
The first qualifying sample records Linux `sysinfo().uptime`; GREEN is allowed
after at least 600 seconds of continuous qualifying sampled conditions. Before
READY, a below-threshold reading, invalid read, or lost external power resets
the interval; samples are not bridged. Initial read failure exits, and six
consecutive later read failures (about one minute at the nominal cadence)
fail closed. Once GREEN, a later voltage dip alone does not clear READY while
external power remains; invalid input clears the cue and requires a fresh
qualification. External-power removal requests BLACK and exits, before or
after READY.

The correction preserved the pure state machine byte-for-byte: both packages'
`runready_logic.h` have SHA-256
`859d752d633fd8b50f78784c20c77ee1c8babc511f1300e51be0e4f3c03bdc8b`. The
recorded diff changes the monitor's helper path from `nxbg` to
`/mnt/sdcard/opentom/power/bin/tomi-rootbg`; threshold, duration, cadence,
reset/latch, and unplug behavior did not change. All 11 synthetic state-machine
tests passed in both work units. The correction's mock Nano-X tests and host
source checks also passed; these establish request construction and host-side
control flow, not rendered pixels.

For the final candidate, the monitor does not call `setenv`: the launcher owns
Nano-X library-path setup, and the forked helper inherits its
`LD_LIBRARY_PATH`. The work-unit launch instructions set
`LD_LIBRARY_PATH=/mnt/sdcard/opentom/lib`; if the helper cannot resolve
`libnano-X.so`, cue invocation fails and the monitor exits fail-closed rather
than showing READY. This is a candidate runtime instruction, not evidence of
the exact current live library path or a deployed run-ready utility.

Validation status for these two work units: `PURE_LOGIC_TESTED`,
`HOST_SOURCE_CHECKED`, and `MOCK_NANOX_TESTED` passed; `ARM_BUILD`,
`TARGET_SMOKE`, and `LIVE_VISUAL_VALIDATION` were not performed; `DEPLOYED`
is false. Treat both run-ready implementations as source-reviewed,
host-tested candidates—not as the current production indicator. The original
`nxbg` cue has separate historical live evidence.
Detailed work-unit and source/test provenance is recorded in
`/mnt/d/Codex/TT3/luce-display-run-ready-coverage-fix-2026-09-25.md`.

## Direct framebuffer versus Nano-X

The reviewed Tomi evidence confirms that applications were developed against Nano-X and that Tomi's startup configured Nano-X over `/dev/fb`. No indexed, Tomi-specific direct-fb trial with identified source, write model, captured pixels, or measured flicker/tearing was located. The bounded review therefore cannot say direct access was attempted, worked, failed, or was the cause of any artifact; it was not found as an adopted technique. Nano-X is the evidenced application path, while framebuffer details remain unresolved. TT1 framebuffer-console experiments and other-device display work are comparative only.

## Build and compatibility boundary

The nxbattery artifacts were built in the pinned legacy ARM/GCC 3.3.4 environment described in [the lab environment reference](../environment/lab-environment.md), dynamically linked to `libnano-X.so` and libc. Their reports include test and ELF metadata; no target deployment occurred for these versions. Preserve the candidate binary hashes above as artifact identities, not proof of installed versions. The run-ready/root-background candidates have host-only validation; no ARM target build or deployment is established. Runtime shell and applet limits are covered by [the runtime ABI reference](tomi-runtime-abi.md).

## Known unknowns

The evidence reviewed here does not establish the active framebuffer geometry or virtual dimensions, bpp/format/channel order/stride/memory/address, framebuffer ioctls or direct-access behavior, exact deployed Nano-X/Microwindows build and library hashes, socket path, installed fonts, touch-event delivery, alpha/transparency semantics, or long-run window coexistence/repaint performance. The run-ready/root-background candidates' target build, runtime library resolution, end-to-end monitor/helper execution, and live whole-desktop visual behavior also remain unvalidated. No LCD model is inferred. A bounded live capture of `fbset -s` plus framebuffer variable/fixed info ioctls, tied to the running kernel and installed software hashes, would close the largest software-stack identity gap; direct-fb testing is not recommended absent a concrete need and a controlled recovery plan.
