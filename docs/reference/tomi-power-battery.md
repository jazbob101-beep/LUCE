# Tomi Power and Battery Reference

## Scope and evidence model

This reference summarizes what the available Tomi source audits, device observations, discharge records, and diagnostic tools establish about battery voltage, external-power detection, charge-status reporting, and related observations. It does not specify TomiDock wiring, the USB-role state machine, battery replacement, or a charging circuit schematic. See [TomiDock Architecture](../architecture/tomidock.md) for physical topology and [Tomi Linux USB Role Kernel Reference](tomi-usb-role-kernel.md) for role arbitration and GPF1/buspower details.

Evidence classes are kept distinct:

- **Live/operator observation:** reported from Tomi or a bench run, subject to the capture and identity limits stated with it.
- **Formal run record:** a preserved archive whose six payload files verify against its embedded manifest. The archive strongly supports a Tomi bench run, but does not independently prove the execution target or bind the run to exact executable bytes. Manifest verification validates archive contents, not unmeasured electrical quantities.
- **Source correlation:** behavior in the OpenTom source tree, not proof that those exact source bytes produced the live kernel.
- **Derived UI/calibration:** calculations from recorded voltage/runtime data, not a physical battery gauge.

The source correlation used by the power and charging audits is OpenTom commit `eb3d2037315bde5da2875b38cba4efba9d47eac7`, in a checkout that was dirty and contained unrelated changes. Exact source-to-live-kernel correspondence is not established. Relevant paths include `src/linux-s3c24xx/drivers/barcelona/bat/bat.c`, `.../drivers/barcelona/buspower/buspower.c`, `.../drivers/barcelona/gpio/gpio.c`, `.../include/barcelona/Barc_Gpio.h`, and the Austin/Acton platform I/O headers.

## Observable power signals

| Signal | Source and representation | What it establishes | Important limits |
|---|---|---|---|
| Battery voltage (`mV`) | `IOR_BATTERY_STATUS` from `/dev/battery`; the driver reads ADC slot 0, filters/scales it, applies its selected calibration, and returns an integer millivolt value. | Linux's calibrated battery-voltage estimate at the ioctl sample. | Not raw ADC counts, current, capacity, state of charge, health, or an exact blackout threshold. Samples are temporally filtered and not atomic with the other GPIO/status reads. Exact live calibration selection is not independently bound to the observed kernel. |
| `CHARGE` / `charge_status` | `BATTERY_STATUS.u8ChargeStatus`, sourced by the audited driver from `ACPWR` and `CHARGING` inputs. The defined values are 0/1/2. | A source-defined combination of external-power detect and the charger's CHRG-status input. | It is not a battery-current measurement. In particular, `1` does not mean current is flowing into the pack; `2` indicates the source's charger-enabled/status signal, not independently measured positive current. |
| `usb_vbus` / `VBUS` | Linux buspower's cached `USB_HOST_DETECT` input; on the audited Austin/Acton map this is active-low GPF1. Buspower polls at 200 ms; `hwstatus` and sysfs consumers expose sampled/cached state. | The logical VBUS/external-power detector was asserted at the relevant software sample. | Not a measurement of 5 V magnitude, stability, current capacity, connector voltage, charge current, data attachment, or USB role. |
| `hwstatus_raw` | Bitmask from `/dev/hwstatus` (`IOR_HWSTATUS`), assembled by the GPIO worker from cached and separately sampled logical inputs. | A packed snapshot of software-visible flags. | It is not a single atomic physical-pin capture. For the audited Acton profile, `0x0080` is the always-active RXD fallback; `0x0002` is cached USB detect; `0x0008` is ignition. `0x008a`, `0x0082`, and `0x0080` can reflect asynchronous observations of aliases of GPF1, not independent rails. |
| Raw GPF1 / GPC7 (`CHRG`) | PowerFlight's read-only `/dev/mem` register readback of GPIO DAT, decoded with the platform map. | Register/latch readback relevant to the logical inputs used by the source. | Not pad voltage or current. GPIO reads are not synchronized with the battery ioctl and do not prove external circuit levels. |
| `LOW_DC_VCC` / peak | `hwstatus` bits `0x2000` / `0x4000` where the board defines and wires the signal. | A source-specific comparator input/latched event when supported. | The audited Acton profile does not define the low-DC pin; the report finds these fields unsupported/structurally zero for Tomi's mapped profile. They must not be treated as a calibrated battery cutoff. |
| USB child presence | USB host topology enumeration, e.g. expected TomiDock child VID:PID `303a:4002`. | A host-side child was visible to the USB stack at that sample. | Does not establish VBUS source/switch state or charging. The rebuilt TomiDock rig can keep D+/D-/GND connected while Tomi VBUS is switched off. |
| USB role | `/sys/devices/platform/tomtomgo-usbmode/mode`, with values such as `USB_STATE_HOST`, `USB_STATE_CLA`, and `USB_STATE_IDLE`. | The kernel's current logical USB authority/role. | Does not establish VBUS, child presence, charge status, or battery current. Role arbitration details belong to the USB-role reference. |

### Battery voltage and calibration limits

The audited driver reports an integer millivolt value derived from ADC slot 0. It samples at 10 Hz and applies a weighted filter `(3*old + sample)/4`; board-specific scaling/calibration is then applied by `bat_get_voltage()`. The user-facing value is therefore a processed software estimate, not the raw ADC word. The source audit did not establish the exact calibration profile active in the current live kernel.

Four battery-only natural-discharge runs were preserved. Runs 2–4 form a short-runtime cluster (about 221.5–228.1 minutes); run 1 is a valid, unresolved longer regime (327.05 minutes). A later 21-anchor table uses the per-anchor median of normalized runs 2–4, from 4150 mV at the representative post-unplug start to 3120 mV at the final durable-record endpoint. It is best described as an **empirical run-relative UI mapping**, not chemical state of charge, capacity, health, or guaranteed remaining time. Confidence is low across roughly 65–100% because run 4's upper curve differs; the one-curve display cannot identify whether the longer run-1 regime is occurring. The endpoints do not mean chemically full/empty, a proven cutoff, or exact blackout.

Historical suspended/ready observations around 3.98 V, 3.92 V, and 3.86 V are mentioned in the migration record, but their original timestamped source records were not located in the bounded evidence reviewed for this migration. They remain isolated historical voltage observations, not calibration anchors or an SOC curve.

## External power, CHARGE, and role context

The source audit maps `charge_status` as follows for the Austin/Acton source profile used to explain Tomi:

| Value | Source interpretation | Safe wording |
|---:|---|---|
| 0 | `ACPWR` false | external-power detector not asserted |
| 1 | `ACPWR` true; `CHARGING` false; software label `CHARGE_STATE_COMPLETED` | external power detected; CHRG input says charger is not enabled; no inference about battery current |
| 2 | `ACPWR` true; `CHARGING` true; software label `CHARGE_STATE_CHARGING` | external power detected; CHRG input says charger is enabled; positive pack current is not measured |

The source calls the status-1 case “completed,” but that name is more specific than the hardware observation warrants. Its reported 200 mA field for status 2 is explicitly a compatibility constant, not a measurement. On Acton, the audited source aliases active-low GPF1 as `ACPWR`, `USB_HOST_DETECT`, and `IGNITION`, explaining why these logical signals can move together while being sampled on different schedules.

The reports describe a bench interval with voltage falling from about 3880 to 3834 mV while `charge_status=1` and `usb_vbus=1`. This is consistent with the source semantics: external power was detected but the CHRG input did not indicate the charger enabled. It does not establish a failed charger or measured negative battery current. Actual input voltage/current and pack current were not recorded in that observation.

Source analysis found that USB `HOST`, `HOST_DETECT`, and `CLA` are all in the powered USB group and request the platform's high-current control; a CLA↔HOST change does not itself change that policy. Whole-system Linux suspend/resume is a distinct path that selects low current. This is source-derived behavior, not proof of the live circuit response. The known design allows USB host data operation with external power and charging policy present, but neither the software signal nor topology guarantees net-positive battery current. No USB Battery Charging compliance claim or exact physical charger-IC identity is made here.

## PowerFlight observability and validation

PowerFlight is a read-only field recorder plus a pure state/event classifier. Depending on version, it records battery and hwstatus ioctls, buspower/role sysfs, USB child identity/topology, USB interface state, read-only GPIO register snapshots, kernel messages, and slower controller/module/interrupt snapshots. Its GPIO DAT is register readback, not an electrical probe. It performs no charging, GPIO, USB-role, PMIC, or kernel-setting writes. Version-specific CSV schemas and test behavior are documented in the individual work units; v001.5.3 retains 70 columns.

The preserved v001.5.3 run archive is `/mnt/d/Codex/TT3/Evidence/powerflight-20260921-170744-7150-00.tar.gz` (38,207 bytes; SHA-256 `67a8c41caa80c09ebe2fe9114bd01ba00b5ab84343b6ef750ccb58c5e52daf8a`); all six payload entries verify against its embedded manifest. Metadata identifies program `tomi-powerflight-v001.5.3`, version `0.1.5.3`, and the VBUS-switch/data-always-connected topology. Its kernel, S3C24xx USB, TomiDock child, battery, GPIO/MMIO, and USB-role records strongly support a physical Tomi bench run. However, no preserved deployment transcript, target-side pre-run digest, archive executable digest, or equivalent provenance binds this run to the exact candidate binaries. The archive proves the integrity and contents of the captured run record, not by itself where it ran or which exact executable bytes ran. The recorded terminal state is:

```text
VBUS=0  CHARGE=0  CHILD=1  USB_STATE_HOST
last_error=0x0  BATTERY_RETURN_CHILD_ATTACHED
```

Within the preserved run record, this is coherent for the rebuilt-rig topology: VBUS is off while the data link and TomiDock child remain present under HOST authority. The candidate recorder source and archive shape align with that observation, and the host unit suite cannot generate these hardware/kernel capture files. This makes a Tomi bench run strongly supported, but the reviewed evidence does not independently authenticate the acquisition path or executed binary. The run record does not prove actual input or battery current, a universal hardware property, or a particular physical VBUS-switch implementation.

v001.5.3 materially corrected v001.5.2's conflation of “battery return” with physical USB cable removal. In the preserved v001.5.2 archive, the child remained present and role stayed HOST after VBUS/charge fell to zero; the run record measures 1535.410 seconds (about 25m35s) from WAIT_A_PRIME to COMPLETE, with child removal shortly before completion. In the preserved v001.5.3 archive, WAIT_A_PRIME to COMPLETE measures about 5.08 seconds. Both are archive measurements; Tomi execution is strongly supported for the v001.5.3 archive but not directly provenance-proven, and neither timing is a guarantee. v001.5.3 accepts either coherent PHYS-2 battery return (expected child + HOST while VBUS is off) or PHYS-1 return (no child + IDLE). The older defect was in the classifier/test topology model, not evidence of a Tomi charging defect.

The lineage is summarized in the reconciliation report. In brief: v001 established the observational A–B–C–B′–A′ recorder; v001.1 hardened qualification, baseline freezing, fail-visible states, and status persistence; v001.2 strengthened independent CHRG/status and topology invariants; v001.3 separated topology availability from core telemetry and fixed truthful readiness; v001.4 corrected exact `USB_STATE_*` authority strings; v001.5 adapted the procedure to the rebuilt topology; v001.5.1 restored fail-closed VBUS-loss handling; v001.5.2 hardened target runtime/documentation; v001.5.3 corrected battery-return classification. The candidate documents record v001.5.3 as not deployed/not run during that candidate work unit; a later, separate archive identifies itself as a completed v001.5.3 run and strongly supports Tomi execution, without binding the run to exact candidate binary bytes. This distinction does not change the v001.5.3 state-machine correction.

## nxbattery display and calibration

nxbattery is an observation/UI tool, not a battery-management controller. It reads the battery ioctl and external-power indication and displays voltage and status. The V002 UI intentionally suppresses percentage and filled gauge when external power is indicated, while keeping raw voltage visible and showing the raw integer `CHARGE` value. This prevents charger-influenced voltage from being presented as battery-only estimated range. V002's `CHARGE` display is not a reinterpretation into current or SOC.

V001 introduced the empirical table/filter/hysteresis display. V002 adds raw `CHARGE` visibility and the expanded presentation; the preserved V002 binary is 12,716 bytes, SHA-256 `a2d14db5628c9687e8c6196284244c719f198cb2ef76fe5e0dd43955d9f0f78f`. The current deployment at `/mnt/sdcard/opentom/ui/nxbattery-v002` is operator-reported and is not independently hash-bound to this preserved package. The calibration remains one-device empirical remaining-run fraction, with the confidence and regime limits above; additional curve work is not a current deployment gate.

## Known unknowns and negative findings

- No inspected source or log provides direct battery current, coulomb count, physical pack chemistry/capacity, temperature, health, or cycle count.
- `CHARGE=1`, external-power detect, VBUS, USB child presence, and `USB_STATE_HOST` are distinct signals; none alone proves that energy is entering the battery.
- No source path was found that disables charging specifically because the role changed from CLA to HOST. The source instead groups those roles for its high-current policy; electrical outcomes remain unmeasured.
- The exact physical cause of a status/CHRG transition, the circuit downstream of GPG8, actual VBUS quality/current, and bootloader/Storage-mode power policy remain unresolved.
- `LOW_DC_VCC` cannot be used as Tomi's calibrated battery threshold on the audited Acton profile; the low-DC input is not defined there.
- USB role and child enumeration can legitimately persist while Tomi's separately switched VBUS is absent; do not infer contradiction or charging from that combination.
- Physical battery chemistry, capacity, exact charger IC, and live-kernel correspondence to the inspected source must remain unknown unless separately established.

## Canonical references and provenance

- [Tomi device profile](../devices/tt3-tomi.md)
- [TomiDock Architecture](../architecture/tomidock.md)
- [Tomi Linux USB Role Kernel Reference](tomi-usb-role-kernel.md)
- [Current State](../../CURRENT_STATE.md)
- Detailed source, run, calibration, and tool provenance is retained in the outside-repository reconciliation report: `/mnt/d/Codex/TT3/luce-bootstrap-tomi-power-battery-reconciliation-2026-09-24.md`.

The report is a migration record, not a substitute for source artifacts or raw captures. Where source-to-binary identity or electrical measurement is unavailable, this reference says so explicitly.
