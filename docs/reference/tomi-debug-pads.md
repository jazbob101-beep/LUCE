# Tomi UART, JTAG, and Debug-Pad Reference

## Scope

This reference records what the preserved Tomi evidence establishes about possible native UART, serial-console, JTAG, and debug/test-pad access. It separates SoC capability and OpenTom source configuration from physical PCB routing and electrical observation. It is not a Tomi board pinout or a probing procedure.

The current evidence does **not** identify any Tomi PCB test pad as UART, JTAG, console, or debugger access. That is a bounded corpus finding, not proof that no such routing exists.

## Evidence model

| Layer | Tomi result |
|---|---|
| `SOC_CAPABILITY` | Tomi’s captured kernel identifies an S3C2412/ARM926 platform. UART and JTAG features documented for this SoC are capabilities only. |
| `SOURCE_PINMUX` | The inspected, dirty OpenTom Austin/type-42 profile assigns dock and GPS UART functions to named SoC pins. Exact correspondence to the running kernel is unproven. |
| `PCB_CONTINUITY` | No Tomi pad-to-SoC-pin or pad-to-peripheral continuity result was found. |
| `ELECTRICAL_MEASUREMENT` | No Tomi test-pad voltage, resistance, pull, or powered/unpowered level measurement was found. |
| `LOGIC_CAPTURE` | No Tomi board-level UART/JTAG logic-analyzer capture was found. |
| `PROTOCOL_OBSERVATION` | Downstream GPS software produced parsed sentence text; raw UART bytes and receiver wire protocol were not captured. |
| `LIVE_CONSOLE` | No native Tomi UART console or prompt was demonstrated. A separate network Telnet shell is not a serial-console observation. |
| `DEBUGGER_HANDSHAKE` | No Tomi JTAG IDCODE, TAP, boundary-scan, halt, or debugger handshake was demonstrated. |
| `PHYSICAL_IDENTIFICATION` | No Tomi pad/test-point identifiers or board orientation map were found in the reviewed Tomi evidence. |
| `SIBLING_COMPARISON` | TT1/Austin-class pad and JTAG investigations are useful context only; their pad locations and electrical findings are not assigned to Tomi. |
| `INFERENCE` / `UNKNOWN` | Physical surfacing, voltage domains, and usable debug access remain unknown. |

“No physical-pad assignment found” must not be upgraded into “no pads exist.”

## Board orientation and pad naming

No reproducible Tomi-specific pad labels, board-side orientation, annotated pad photographs, or pad catalog were discovered in the reviewed corpus. Consequently this document intentionally supplies no numbered pad diagram. TT1 labels such as `B128–B139`, `F016–F020`, and `J14/B132` belong to TT1 evidence and are not Tomi pad names.

## Debug-access overview

| Access path | Strongest surviving evidence | Classification | What is not established |
|---|---|---|---|
| S3C2412 UART controllers | Tomi kernel transcript reports UART0/1/2 controller registration and MMIO/IRQ tuples. | `LIVE_DEVICE_NODE` / kernel log; controller inventory | External pads, selected console, baud/framing at the pins, or physical route |
| Dock UART | Austin profile source sets `dockdev="ttySAC0"`, TX `GPH2`, RX `GPH3`; runtime kernel log reports `gpio_hw_init: UART 'ttySAC0' => 0`. | `SOFTWARE_CONFIGURED` and source-correlated; not PCB-mapped | Continuity to a Tomi connector/test pad, electrical levels, or live dock UART protocol |
| GPS UART | Austin source maps GPS TX/RX to `GPH4/GPH5`, flow control to `GPG9/GPG10`, and selects `ttySAC1`; the Aug. 24 transcript observes `/dev/ttySAC1` and `/dev/gpsdata -> /dev/ttySAC1`. | `SOFTWARE_CONFIGURED`, `LIVE_DEVICE_NODE`; downstream GPS protocol-correlated | Physical pad mapping, raw serial waveform/bytes, independently measured baud/framing, or console use. See [Tomi GPS and glgps](tomi-gps-glgps.md). |
| UART2 / other serial roles | UART2 appears in the Tomi kernel controller inventory. | Controller exists in that captured kernel | Board-level assignment or use on Tomi |
| Linux serial console | Captured kernel command line is `root=/dev/mmcblk0p2` and contains no `console=` argument. The preserved transcript is a Telnet capture of `dmesg`, not serial output. | No native console established | A console selected by another mechanism, a different software specimen, or a physical console route |
| Bootloader serial console | Tomi reports bootloader family/version `s3c24xx` 5.5279; the analyzed matching update-package work did not establish a live UART console or Tomi pad route. Its bounded ingress audit found no grounded serial downloader. | Static behavior unresolved; no runtime console observation | Whether the active NOR loader emits UART diagnostics, listens on serial, or exposes a shell |
| ARM JTAG / boundary scan | S3C2412 documentation and TT1 comparative work describe SoC TAP capability. | `SOC_CAPABILITY` / `SIBLING_COMPARISON` | Any Tomi TAP pad, complete scan chain, strap state, IDCODE, or debugger access |
| USB-attached CP210x serial | A Tomi experiment enumerated a CP2101/CP2102 USB-to-UART bridge under Linux OHCI; its open/configure request timed out and the helper reported it could not enable the adapter UART. | `TESTED_NEGATIVE` for that USB adapter path | Native Tomi UART-pad access; it was not a UART header or board-pad test |

## UART

### SoC and source assignments

The captured Tomi kernel log reports these controller registrations (kernel-reported MMIO and IRQ values, not PCB coordinates):

| Kernel controller | Reported MMIO | IRQ | Tomi-specific role evidence |
|---|---:|---:|---|
| `s3c2412-uart.0` / `ttySAC0` | `0x50000000` | 70 | Austin source calls it the dock device and assigns dock TX/RX to GPH2/GPH3. |
| `s3c2412-uart.1` / `ttySAC1` | `0x50004000` | 73 | Austin source calls it GPS and assigns GPS TX/RX to GPH4/GPH5, CTS/RTS to GPG9/GPG10. The live transcript uses this endpoint for GPS software. |
| `s3c2412-uart.2` / `ttySAC2` | `0x50008000` | 76 | Controller is reported, but no Tomi board function or external pad is established. |

The `ttySAC*` names and pin assignments are software/source evidence. They do not locate a physical pad. The OpenTom source checkout was dirty and the relevant profile files were not established as tracked blobs at the recorded source `HEAD`; treat the mappings as `SOURCE_PINMUX`, not exact-image proof.

### Physical pad findings

No Tomi physical pad assignment reaches `PCB_MAPPED`, `ELECTRICALLY_OBSERVED`, or `DECODED_SERIAL`. There is no preserved continuity table, Tomi pad label, scope/logic capture, idle-level record, resistance measurement, or adapter connection tied to a native Tomi UART pin.

The GPIO/power source audits identify software names such as GPF1, GPC7, GPG5/GPG8, GPE11/GPE12, and GPF7. Those are source-level pin and controller-policy facts, not debug-pad findings. Existing readback analyses explicitly distinguish SoC register state from electrical pad/net voltage. No actual Tomi external pad measurement is thereby implied.

### Serial-console findings

The captured Linux command line is `root=/dev/mmcblk0p2`; the saved command-line string does not select `console=ttySAC*`. The kernel messages in the preserved transcript were retrieved with `dmesg` over a network/Telnet session. No native serial boot log, login prompt, getty attachment, interactive UART shell, or bootloader prompt is established. Absence of `console=` in this one captured command line does not prove that every Tomi software version or boot path lacks serial output.

The bootloader audit’s “serial downloader NOT FOUND” is a bounded static ingress result, not proof of silence on every UART or every bootloader branch. Static diagnostic support, if present in some loader code, would still not establish physical pin access or runtime output.

### Negative results

- The Aug. 23/24 “USB serial smoke test” did not test Tomi’s native UART pins. It attempted to use a USB-attached CP210x bridge at `/dev/ttyUSB`; the preserved run reports that the node could not be opened and the bridge could not enable its UART after a USB request timeout. It is a failed USB adapter path, not evidence for or against a board test pad.
- The same terminal evidence separately shows a CP2102 USB device and `/dev/ttyUSB0` enumeration during earlier probes. Those USB-serial endpoints are not `/dev/ttySAC1` and do not map native UART pins.
- GPS text captured downstream is not raw wire capture. It does not establish receiver-originated framing or a physical pad.
- No Tomi JTAG signal assignment, `JTAG` header, IDCODE, TAP chain, or OpenOCD handshake was found.

## JTAG

The S3C2412 family has a documented ARM TAP, and the TT1 JTAG work examined Samsung reference documentation including the conditional package-ball set `nTRST H15`, `TDI J12`, `TDO J13`, `TCK J14`, `TMS J15`, and `RTCK J16`. Applicability of those package locations to Tomi’s exact package/revision and, especially, routing to Tomi PCB test points is not established here. The Samsung SMDK connector topology is a reference-board design only.

The TT1 investigation’s candidate areas (`B128–B139`, `F016–F020`, and a dense back-side bus field) are TT1 identifiers and low-confidence hypotheses, not Tomi candidates. TT1’s J14/B132 electrical/software-causality results are also tied to that TT1 investigation. No such signal names, coordinates, voltage, continuity, or CPU-less donor behavior are transferred to Tomi.

Tomi status: `SOC_CAPABILITY` is contextual; `SOURCE_PINMUX_KNOWN` for Tomi TAP is not; `PCB_CONTINUITY_KNOWN` is not; `PAD_ELECTRICAL_MATCH` is not; `JTAG_HANDSHAKE_PROVEN` is not. No safe Tomi JTAG pinout is available.

## Other test pads

Tomi battery/USB audits discuss several named SoC GPIO signals and register addresses, including power, VBUS, charger, and USB-suspend-related functions. These sources can establish software-configured logical pins, but no reviewed record correlates any one of them to a named physical test pad or measures its voltage. They are not a general board map and should not be used to probe unidentified contacts.

## Pin-mux and ownership constraints

OpenTom’s Austin/type-42 software profile assigns multiple functions to S3C2412 pins. Such assignments are conditional software configuration, not proof the pin reaches a user-accessible test point. A pin’s SoC alternate-function capability does not establish which mode is active at reset, during bootloader execution, in Linux, or during suspend. Dock, GPS, GPIO, and possible JTAG roles must not be assumed simultaneous.

The canonical GPS reference owns the GPS subsystem interpretation. This page retains only the physical-debug boundary: `GPH4/GPH5` and `GPG9/GPG10` are source-declared GPS pins; no Tomi board-level continuity or pad measurement establishes them physically.

## Cross-device comparisons

| Evidence | Relationship | Permitted use | Not transferable |
|---|---|---|---|
| TT1 `JTAG_Investigation` and `J14-investigation` | Same broad TomTom/S3C2412/Austin-class context, but separately catalogued TT1 hardware | Shows how SoC capability and low-confidence pad hypotheses were kept distinct; useful negative/software-causality context | TT1 board orientation, pad numbers, continuity, CPU-less measurements, voltage, or JTAG availability |
| TT1 K17/GPH7 audit and UART/CBEE experiments | TT1-specific source/binary/live work | Methodological and source-family context | Tomi GPIO use, console, pad routing, or UART activity |
| TJ Atlas III source/symbol work | Different SoC/platform family | Demonstrates separate SiRF/Atlas UART/platform implementation | All Atlas addresses, pins, board routes, and debug conclusions |
| Samsung SMDK2412 reference schematic | Reference board | SoC signal names/reference circuitry only | Tomi connector, pull-ups, voltage, or routing |

Even a matching SoC or TomTom software profile does not establish identical PCB routing.

## Safe future-probing constraints

This is documentation of existing evidence, not authorization for a new experiment. No unidentified pad should be connected to an active UART/JTAG adapter based only on appearance, a SoC pin-mux table, a sibling board, or a voltage that “looks UART-like.” Existing evidence does not establish Tomi pad voltage domains, 5 V tolerance, open-drain behavior, pull networks, VREF, or ground labels. Do not import SMDK or TT1 electrical assumptions into Tomi.

Any future physical work requires a separately scoped, evidence-led plan and direct identification for the exact Tomi board. This reference itself authorizes no probing, soldering, powering, reset, halt, flash, or device interaction.

## Known unknowns

- Whether Tomi exposes UART, JTAG, boundary-scan, or manufacturing-test signals at any accessible pad.
- Any Tomi pad labels, orientation, map, SoC-ball continuity, component-terminal continuity, or hidden-layer routing.
- Electrical idle levels, pull resistors, voltage domain, VREF, grounding, and tolerance of candidate points.
- Whether the active 5.5279 bootloader emits serial diagnostics or accepts serial input on any physical route.
- Whether any Linux version selects a UART console through a mechanism not represented by the saved `console=` command line.
- The physical protocol, measured baud/framing, and raw bytes on the GPS UART wire.
- The exact built/running kernel correspondence to the inspected dirty Austin source checkout.
- Any Tomi JTAG scan-chain state, IDCODE, boundary-scan mode, or debugger response.

## Canonical references

- [Tomi device profile](../devices/tt3-tomi.md) — device and platform identity; does not assert debug-pad accessibility.
- [Tomi GPS and `glgps`](tomi-gps-glgps.md) — GPS UART software endpoint and explicit raw-wire limitations.
- [Tomi boot chain](../architecture/tomi-boot-chain.md) and [bootloader image reference](tomi-bootloader-image.md) — bootloader specimen and static-analysis boundaries.
- [Tomi Linux USB-role kernel reference](tomi-usb-role-kernel.md) — Linux USB-role interfaces, not serial-pad mapping.
- [Tomi disaster-recovery runbook](../runbooks/tomi-disaster-recovery.md) — recovery boundaries; UART/JTAG is not a proven rescue path.
- [Evidence Index](../../evidence/INDEX.md) — selected durable evidence anchors.

## Provenance

The local reconciliation record `/mnt/d/Codex/TT3/luce-bootstrap-tomi-debug-pads-reconciliation-2026-09-24.md` inventories the searched Tomi and cross-device work units, pad/photo evidence, source and runtime observations, failed probes, and unresolved claims. Raw evidence and tooling remain in their original investigation locations and were not copied into LUCE.
