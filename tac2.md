# TAC-II — Time-to-Amplitude Converter (TDC) in the DGS Master Trigger

_Source: `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/TAC.docx` — J.T. Anderson, 2016-02-24_
_Supporting: `LabNotes/20210314_TDC.ods`, `LabNotes/Jitter Analysis/`, `LabNotes/20210831_lab_notes.odt`_

---

## Overview

The TAC-II is a **Time-to-Digital Converter (TDC)** implemented in FPGA firmware within the DGS Master Trigger (MTRG). It measures the arrival time of a NIM input signal (e.g. beam RF, external pulser) relative to the trigger timestamp, providing sub-nanosecond timing resolution for TAC-based coincidence measurements.

- **Resolution:** ~30–50 ps (vernier interpolation)
- **Coarse clock:** 250 MHz (4 ns per count), derived from 4-phase multiplication of the 50 MHz main trigger clock
- **Fine interpolation:** 4 vernier chains (0°, 90°, 180°, 270° phases), each 64 taps × ~50 ps/tap
- **Output:** 15-word event packet, sent alongside trigger data

---

## Architecture

### Clock Generation

The main 50 MHz trigger clock is multiplied to **250 MHz** inside the FPGA and distributed in **four phases**, each 90° apart (nominally 1 ns apart). Measured chain delays in the Feb 2016 firmware:

| Chain | Phase | Delay to vernier[0] |
|-------|-------|---------------------|
| A | 0° | 1.670 ns |
| B | 90° | 1.826 ns |
| C | 180° | 1.750 ns |
| D | 270° | 1.742 ns |

Phase skew = 156 ps. Temperature-dependent variation expected in the tap delays.

### Vernier (Delay Line) TDC

Each chain uses FPGA carry-chain elements as a **64-tap delay line**. When the NIM input fires, the 64-bit thermometer code is latched into flip-flops. A **data compression pipeline** converts the 64-bit code into a **6-bit position value** (0–63) indicating how far the edge propagated. Time per tap: nominally **50 ps**.

### Coarse Counter

A **16-bit counter** at 250 MHz runs in parallel with each vernier chain. It rolls over every **262.144 µs** (65536 × 4 ns). The coarse count is latched with the vernier data and synchronized to the main 48-bit timestamp via Imperative Sync.

### Data Collection State Machines

Three cooperating state machines coordinate data capture:
1. **FIFO READER** (×4, 100 MHz) — monitors each vernier FIFO, reads when data available
2. **TRIG_MON_COLLECT** — watches trigger algorithms; when selected algorithm fires, asserts `WANT_NEXT_TDC` and collects trigger message
3. **TDC_AUTOSAMPLE** — collects TDC data from all four FIFO READERs after `WANT_NEXT_TDC`

A variable delay elapses between `WANT_NEXT_TDC` and when `TDC_AUTOSAMPLE` finishes collection. The **pipeline delay is 350 ns** — any chain with differential > 350 ns vs TDCtsLo is stale and invalid.
