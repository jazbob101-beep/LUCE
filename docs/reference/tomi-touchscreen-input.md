# Tomi Touchscreen and Input Reference

## Scope and evidence model

This reference covers input acquisition, calibration, coordinate conversion, and application delivery for TT3 “Tomi”. It does not identify a replacement panel, specify the display, or equate source configuration with the exact running image. Rendering and framebuffer behavior remain in [Tomi Display and Nano-X](tomi-display-nanox.md).

Evidence labels used here:

- **LIVE_OBSERVED** — a preserved Tomi runtime transcript or process snapshot.
- **CAPTURED_CONFIG** — configuration text preserved from a Tomi startup context, without proof every setting was active at capture time.
- **SOURCE_CORRELATED** — inspected OpenTom source describes a driver/path whose exact identity with Tomi’s running kernel or userspace binary is unproven.
- **APPLICATION_ASSUMPTION** — dimensions or behavior assumed by an application, not measured hardware properties.
- **COMPARATIVE_ONLY** — evidence from TT1 or another device family; not transferable to Tomi.
- **UNRESOLVED** — the reviewed evidence does not establish the claim.

## Input-stack overview

The strongest surviving model is:

```text
physical touch hardware / controller: UNKNOWN
        ↓
S3C24xx ADC touchscreen channels (OpenTom source-correlated)
        ↓
TomTom touchscreen input driver: ABS_X, ABS_Y, ABS_PRESSURE (source-correlated)
        ↓
/dev/input/event0: opened by /bin/ttn in one preserved process snapshot (live observation)
        ↓
tslib raw input → pthres → dejitter → linear (captured configuration/source correlation)
        ↓
calibrated sample → Nano-X mouse coordinates/button (source-correlated)
        ↓
application event handling: not demonstrated on Tomi
```

The arrows do not establish one end-to-end captured touch. There is no preserved Tomi input-event trace, calibration output, or Nano-X button/mouse event capture.

## Physical touchscreen evidence

| Claim | Evidence and classification |
|---|---|
| Tomi runs software identifying TomTom touchscreen components | **LIVE_OBSERVED:** captured kernel log prints “TomTom GO Touchscreen Driver” and “TomTom GO Touchscreen Input Driver” banners. This identifies driver-family messages, not a physical controller. |
| Touch uses analog ADC channels in the inspected OpenTom driver family | **SOURCE_CORRELATED:** `drivers/barcelona/tsinput/tsinput.c` consumes `ADC_TS_X`, `ADC_TS_Y`, and `ADC_TS_DOWN`; the ADC source sets touchscreen channel sampling. It is not a board-level measurement. |
| Archived touch-calibration record | `/mnt/d/Codex/TT3/TT3_Archival_Toolkit/TT3_Archival_Runbook.md` records `81 936 132 860, @3300mV` and says touchscreen passed an initial functional test. **CAPTURED_DIAGNOSTIC_RECORD:** raw diagnostic output, format, date of calibration, coordinate domains, and consuming component are not preserved there. |
| Resistive/capacitive technology, four-/five-wire construction, controller/ADC part, connector, panel model, physical dimensions, sample behavior | **UNRESOLVED.** Neither the logs nor reviewed source identifies these physical properties for Tomi. ADC/pressure software alone is insufficient to assert a panel topology. |
| 480×272 panel geometry | **APPLICATION_ASSUMPTION** in some Nano-X candidate layouts and legacy source branches, not an independently measured live display or touch range. See the display reference. |

## Kernel input path

### Runtime evidence

The preserved Tomi kernel transcript at `/mnt/d/Codex/TT3/glgps-forensics/tomi-telnet-log.txt` (SHA-256 `0bc2d68d9211208c557330dc9a0420affa8d7bc245d40e00aa4107d8d1e73d24`) reports both the older “TomTom GO Touchscreen Driver” and the 2007 “TomTom GO Touchscreen Input Driver” (around transcript lines 3079–3080; repeated in later captured boot logs). **LIVE_OBSERVED:** those driver banners occurred in that kernel session. They do not disclose event-node assignment or establish source identity.

The preserved process snapshot `/mnt/d/Codex/TT3/TT3-capture-20260804-163938/filesystem-copy/restart.dat` (SHA-256 `0db7d0d77e97d40983ee7c0682499c75cd31a3b562122de985770230cfd60d37`) identifies PID 369 as `/bin/ttn`, version `9.061.576030 (s3c24xx)`, and lists file descriptor 5 as `/dev/input/event0`. **LIVE_OBSERVED:** that TomTom process had opened event0 at the recorded crash snapshot. It does not show event samples, input capabilities, whether event0 was exclusively touch, or stable numbering across boots/images.

Extracted initramfs trees contain `/dev/input/event0` and other event-node entries, but these are static filesystem nodes/placeholders; they do not identify the registered live device. No Tomi `/proc/bus/input/devices`, `evtest`/equivalent capability output, live `EV_ABS` range, `BTN_TOUCH` record, or raw sample was found.

### Source-correlated driver behavior

The inspected OpenTom checkout is rooted at `/home/jazbob/opentom`, Git `HEAD` `eb3d2037315bde5da2875b38cba4efba9d47eac7`; the worktree is dirty. The following file hashes pin the inspected files, not Tomi’s running kernel:

| Source | SHA-256 | Source-grounded behavior |
|---|---|---|
| `src/linux-s3c24xx/drivers/barcelona/tsinput/tsinput.c` | `08e2555dfad4b4063726cccbde50d4fceab07a70682da0261158d090dba765d9` | Registers an input device named `TomTom GO Touchscreen Input Driver`; advertises `EV_ABS`/`EV_KEY`, `ABS_X`, `ABS_Y`, `ABS_PRESSURE`, and `BTN_TOUCH`; X/Y/pressure ranges are 0–1023. Its tasklet reports X/Y and pressure (`1023` down, `0` up), followed by `input_sync`. It does not call `input_report_key(BTN_TOUCH)` in the inspected path, despite advertising that capability. |
| `src/linux-s3c24xx/drivers/barcelona/adc/adc.c` | `3f77327ff96a7e36c1a3522c2a95dc26c4e0ccd17886b234b8816a19ff1aadb9` | Selects `ADC_TS_X/Y/DOWN` channels and configures touchscreen-related ADC sampling. |
| `src/linux-s3c24xx/drivers/barcelona/ts/ts.c` | `a8171a2cc291ce9cbe4c0346f72fab9e0a1b4c2735479fdc5352c126c8348097` | Separate legacy character-device interface. Has a source-default matrix and converts raw samples using TFT-type-dependent 320×240 or 480×272 clamp dimensions, a 15-pixel offset, and X inversion. It exposes calibration/raw-mode ioctls. This is not the tslib `pointercal` format and is not proof these defaults were active on Tomi. |

In `include/barcelona/Barc_adc.h`, this source profile sets `ADC_RATE_TS=ADC_MRATE=50` and `TSINPUT_PRESSURE_TRIGGER=100`; comments say touchscreen sampling was reduced to 50 Hz for larger signal capacitors on Bergamo/Boston. This is **SOURCE_CORRELATED**, not a measured Tomi event rate or a Tomi board identification. The input driver source uses a pressure threshold, averages nearby samples, and retains a 1/10-second pen-up debounce interval. No target timing measurement corroborates it.

The source checkout’s dirty state and lack of exact kernel build/source identity prevent these implementation details from being treated as exact running-kernel behavior.

## tslib configuration and calibration

### Captured startup configuration

The preserved startup-context text in the Tomi Telnet transcript exports:

```sh
TSLIB_CONSOLEDEVICE=none
TSLIB_FBDEVICE=/dev/fb
TSLIB_TSDEVICE=/dev/input/event0
TSLIB_CONFFILE=$DIST/etc/ts.conf
TSLIB_PLUGINDIR=$DIST/lib/ts
TSLIB_CALIBFILE=$DIST/etc/pointercal
```

It also contains `if [ ! -f $TSLIB_CALIBFILE ]; then ts_calibrate; fi`. This is **CAPTURED_CONFIG**: it shows the intended paths and conditional calibration command, not proof that the command ran, that a `pointercal` file existed, or that these environment variables were active for the recorded Tomi process.

The preserved OpenTom skeleton configuration `/home/jazbob/opentom/src/opentom_skel/etc/ts.conf` (SHA-256 `f94487af972747f90bedfe89dca1536d5fc2bef1901f333359aeebe92d1fb3e6`) contains:

```text
module_raw input
module pthres pmin=1
module dejitter delta=100
module linear
```

The source build notes select `TSLIBMOUSE=Y`, `EVENTMOUSE=N`; the inspected Nano-X `mou_tslib.c` opens the configured node (falling back to `/dev/input/event0`), calls `ts_config`, reads `ts_sample`, passes sample X/Y to Nano-X and treats nonzero pressure as left-button-down. This establishes a plausible configured software chain in the inspected source tree, not the identity of Tomi’s deployed Nano-X/tslib binaries or a delivered touch event. The bundled `tslib-1.0` source archive is comparative build material; deployed Tomi tslib version is **UNRESOLVED**.

### Calibration artifacts and interpretation

An archival TT3 runbook records the four-value tuple `81 936 132 860, @3300mV` under “Touch calibration” and says touchscreen passed initial functional testing. Treat this as a **CAPTURED_DIAGNOSTIC_RECORD**, not a decoded transform: the original screen/output, tuple format, provenance/date, raw/output domains and consuming code are missing. It cannot be identified as tslib `pointercal`.

No Tomi `pointercal` contents, six/seven-coefficient tslib record, calibration-session output, or target-side `ts_calibrate` result was located in the bounded preserved-file review. Accordingly there are **no verified live tslib calibration coefficients** to publish.

The legacy `ts.c` static matrix (`An=-363, Bn=0, Cn=360416, Dn=0, En=258, Fn=-12676, Divider=1000`, source ranges 0–1023) is a **SOURCE_DEFAULT** for a distinct legacy interface. It is not evidence of Tomi’s tslib calibration, physical panel dimensions, ADC resolution, or the transform used by `nano-X`.

The source `ts_calibrate.sh` wrapper pauses `nxmenu`, quits `nano-X`, invokes `ts_calibrate`, then resumes `nxmenu`. It is source tooling only. The actual target binary identity, whether it ever ran on Tomi, and any resulting file are **UNRESOLVED**. No calibration was performed for this reference.

## Coordinate domains and Nano-X delivery

| Domain | What is established | What remains unknown |
|---|---|---|
| Physical contact → controller/ADC | OpenTom source correlates touch sampling with S3C24xx ADC channels. | Physical construction, controller, orientation, raw electrical values. |
| ADC → Linux ABS | Inspected `tsinput` profile declares X/Y/pressure 0–1023 and pressure-gated reports. | Whether this exact driver binary ran; actual axis polarity, extrema, clipping, or observed values. |
| Linux event node | `ttn` held `/dev/input/event0` in one crash snapshot; startup config also names event0. | Runtime device name/capabilities and event0 stability. |
| tslib output | Configured chain includes raw `input`, pressure threshold, dejitter, and linear modules. | Active modules and coefficients on Tomi; calibrated output range. |
| Nano-X pointer | Inspected tslib mouse driver maps sample X/Y to pointer coordinates and pressure to left button. | Whether Tomi’s active Nano-X used this driver and whether it generated or delivered events. |
| Application/window | `nxdclock` appears in a preserved Tomi process capture; Nano-X clients exist in project work. | Any Tomi application’s touch response, hit testing, focus/capture behavior, or event masks. |

No origin, axis direction, scaling, or screen-coordinate range should be inferred across the unknown boundaries. The `480×272` application layout assumption is not a raw-touch range. **No target Nano-X mouse/button event or application response has been recorded.**

## Applications and physical buttons

- **nxdclock / nxbattery / nxbg:** their presence or source drawing behavior establishes graphical clients/cues, not touch input consumption. Existing nxbattery and run-ready work did not validate touch delivery. Classify input-consumer behavior as **UNKNOWN**.
- **ThomasTouch:** TT1 ThomasTouch Desktop and the TT3 transplant are application/Nx GPS packaging efforts. The TT3 package contains `nxlaunch`, `nxgps`, config and icon payloads only; no kernel driver, tslib, calibration or input probe. No evidence found that it contributes Tomi core-input knowledge or validated touchscreen handling. **OUT_OF_SCOPE** for the acquisition stack; retain as a separate UI/application lineage.
- **Rescue “three-tap” trigger:** the project rescue trigger samples raw GPF0 in early Linux; it is **EARLY_BOOT_RAW_GPIO**, not a touchscreen or tslib gesture. See [Tomi Disaster Recovery](../runbooks/tomi-disaster-recovery.md) for operational semantics. Normal power-button policy belongs with the power/battery reference, not the touchscreen path.

## Negative findings and known unknowns

- No direct physical identification of Tomi’s panel, touch controller, ADC part, connector or wiring topology.
- No live `/proc/bus/input/devices` inventory, event capability dump, raw event trace, measured axis ranges, pressure sample, or stable event-node map.
- A 4-value calibration tuple is preserved in the archival runbook, but its format, provenance, semantics and active use are unknown; no verified Tomi `pointercal` file or tslib coefficients were found.
- No deployed tslib/Nano-X binary identity tied to the configuration, and no source-to-running-kernel identity.
- No measured touch rate, raw coordinate extrema/origin, axis direction, transform output, event latency, or suspend/resume behavior.
- No target touch-to-Nano-X event or graphical application response; no established multitouch or capacitive-gesture behavior.
- TT1 probes, configurations, and calibration assumptions are comparative only; no panel, node number, coefficient, or behavior is transferred to Tomi.

## Canonical references and provenance

- [Tomi device profile](../devices/tt3-tomi.md) — platform identity and existing touch-driver log boundary.
- [Tomi Display and Nano-X](tomi-display-nanox.md) — framebuffer, rendering, and Nano-X server/client context.
- [Tomi Runtime ABI](tomi-runtime-abi.md) — live target command/tool compatibility.
- [Tomi Power and Battery](tomi-power-battery.md) — power-button and power semantics.
- [Tomi Disaster Recovery](../runbooks/tomi-disaster-recovery.md) — raw-GPIO rescue trigger semantics.
- The external reconciliation report at `/mnt/d/Codex/TT3/luce-bootstrap-tomi-touchscreen-input-reconciliation-2026-09-24.md` inventories investigations, negative results, provenance, and future evidence-index candidates. Raw evidence and tooling remain in their original locations.
