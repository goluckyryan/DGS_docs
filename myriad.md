# MγRIAD — Multipurpose γ-Ray Interface to Auxiliary Detectors

**Full name:** Multipurpose γ-Ray Interface to Auxiliary Detectors  
**Source:** `DGS_SVN/dgs/MyRIAD/Documentation/MyRIAD Abridged User Notes.pdf` (v0.0, 2018-04-27, J. Anderson)  
**Also:** `DGS_SVN/dgs/MyRIAD/Documentation/MYRIAD_Module_Specification.pdf`  
**Wiki PDFs:** `https://wiki.anl.gov/wiki_gsdaq/images/4/40/MyRIAD_User_Manaual.pdf`

---

## Purpose

The MγRIAD bridges auxiliary detectors (NIM/ECL-based) to the DGS/GRETINA TTCL trigger system. It:
- Receives the TTCL timestamp from the Master Trigger via RJ45 (Cat5e, same as digitizer)
- Propagates DGS/GRETINA timestamps to auxiliary VME-based DAQs
- Sends trigger messages back to the Master Trigger when the auxiliary detector fires
- Provides local coincidence logic between local detector and auxiliary trigger
- Interfaces legacy FERA ADC systems via ECL

Connected to **link R or U** of the Master Trigger (see `MTRG_connectors.md`).

---

## Front Panel Connectors

### RJ45 — TTCL Link
- **Not Ethernet** — Cat5e cable to DGS/GRETINA trigger module only
- Carries TTCL (Trigger Timing and Control Link) — same protocol as digitizer RJ45
- LEDs on connector indicate SERDES link lock state

### JTAG
- Direct FPGA access via Xilinx JTAG programmer

### ECL CTL Header (10-pin, 2 differential inputs + 3 differential outputs)
- Pinout compatible with FERA ADC system cables
- Firmware-defined; current DGS build uses ECL CTL outputs as diagnostics:
  - **FERA FULL** pair → copy of multiplexed 50 MHz FPGA clock (locked = TTCL sync OK)
  - **FERA ACK** pair → copy of NIM input 1 (level translator)

### ECL I/O Header (16 differential ECL signals)
- Default: 16 receiver inputs (100 Ω termination per differential pair)
- Assembly positions allow installing driver chips instead (reconfigurable)
- All signals available to firmware

### NIM I/O — 8 inputs, 4 outputs
Layout: two groups of 4 inputs (I), two groups of 2 outputs (O).

#### NIM Input Functions
| Input | Function |
|-------|---------|
| **NIM In 0** (upper left) | **Local system trigger input** — latches timestamp on each edge |
| **NIM In 1** | Local coincidence input — starts coincidence timer after NIM In 0 edge; asserts coincidence if NIM In 1 fires before timeout |
| NIM In 2–7 | General purpose — counted only (no trigger function as of 2018-04-27) |

- All NIM inputs connected to 16-bit edge counters (sampled at 100 MHz)
- All NIM input states regularly sent to Master Trigger over SERDES

#### NIM Output Functions
| Output | Selection | Signal Options |
|--------|-----------|---------------|
| **NIM Out 0** | Gating reg bits 3:2 | `SYNC_ERROR_FLAG` / `AUX_DETECTOR_TRIG` / `SYNC_CAPTURE_FLAG` / `TS_LATCH_BUSY` |
| **NIM Out 1** | Gating reg bits 6:4 | `TTCL_TRIG_FLAG` + others |
| NIM Out 2–3 | Firmware-defined | — |

---

## Front Panel LEDs (3×3 array)

```
P1  P2  V4
F3  F5  S6
S7  S8  S9
```

| LED | Label | Meaning |
|-----|-------|---------|
| 1 | P | +5V VME power present (blue) |
| 2 | P | DC-DC converter subsidiary voltages OK (green) |
| 4 | V | VME access activity (flashes on each VME access) |
| 3 | F | FPGA configuration in progress (blinks) |
| 5 | F | Firmware-specific main FPGA indicator (currently unused) |
| 6 | S | Blinks when internal coincidence logic satisfied |
| 7 | S | Blinks on NIM input 1 leading edges |
| 8 | S | Blinks on local detector TRIGGER IN edges (NIM In 0 or ECL FERA WSI, firmware-selectable) |
| 9 | S | Blinks on NIM input 7 leading edges |

---

## DGS Usage

In DGS, MγRIAD is connected to **link R or U** of the MTRG:
- Receives TTCL timestamps → propagates to auxiliary VME DAQs
- Sends auxiliary detector trigger messages back to MTRG
- Local NIM input 0 = aux detector trigger (e.g. ancillary detector, tape station)
- Local NIM input 1 = coincidence gate signal

Coincidence timer is programmable via register — window defines valid auxiliary trigger window relative to NIM In 0.

---

## Related Files
- `MTRG_connectors.md` — links R and U on MTRG where MγRIAD connects
- `ttcl.md` — TTCL protocol MγRIAD uses for trigger communication
- `wiki_gsdaq.md` — MγRIAD mentioned in DAQ system overview

## SVN Location
`DGS_tools_pack/DGS_SVN/dgs/MyRIAD/`

---

*Created: 2026-04-05 (from SVN MyRIAD Abridged User Notes PDF)*
