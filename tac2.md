# TAC-II — Time-to-Amplitude Converter (TDC) in the DGS Master Trigger

_Source: `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/TAC.docx` — J.T. Anderson, 2016-02-24_
_Supporting: `LabNotes/20210314_TDC.ods`, `LabNotes/Jitter Analysis/`, `LabNotes/20210831_lab_notes.odt`_

> ⚠️ **WARNING:** The FPGA firmware has been significantly modified since the 2016 TAC.docx was written. **The VHDL source code is the ground truth.**
>
> **For the authoritative FPGA-verified packet format and timing details, see:**
> **`deep_fpga_MTRG_MAIN.md` — § "TAC-II / TDC" — verified against VHDL 2026-04-04**
>
> Key differences from this doc:
> - Packet word count: `NUM_TDC_WORDS` and `NUM_TRIG_WORDS` are **configurable** (not fixed at 15 words)
> - MON7B packet format: Word 0 = `0xAAAA`, Words 1..N = TRIG_MON data, Word N+1 = length+user data, Words N+2..end = TDC vernier words
> - Timeout window: **960 ns** (updated 2025-07-22 from 640 ns)
> - Per-phase timing adjustments (+3/0/+1/+2 ns) may have changed with Vivado port
> - `receiver.h` is the most reliable reference for what software actually receives (repacked 10-word format)

---

## Overview

The TAC-II is a **Time-to-Digital Converter (TDC)** implemented in FPGA firmware within the DGS Master Trigger (MTRG). It measures the arrival time of a NIM input signal (e.g. beam RF, external pulser) relative to the trigger timestamp, providing sub-nanosecond timing resolution for TAC-based coincidence measurements.

- **Resolution:** ~18–40 ps σ (vernier interpolation, synchronous tests) ✅ verified 2026-04-09 — TAC.docx §Tests: synchronous σ = 0.035 ns (first cable), 0.040 ns (+1 ns cable), 0.018 ns (+2 ns cable); per-tap adjustment = 50 ps per TAC.docx §Calculation; VHDL MAXDELAY=100ps, actual ~70 ps/step per `tdc_unit_cont.vhd:L160`. Asynchronous σ = 2.4 ns (dominated by input signal jitter, not TDC resolution).
- **Coarse clock:** 250 MHz (4 ns per count), derived from the 50 MHz main trigger clock via DCM CLKFX (×5 multiply: `CLKFX_MULTIPLY=5, CLKIN_PERIOD=20ns`), then distributed in 4 phases (0°/90°/180°/270°) via a second DCM ✅ verified 2026-04-08 — `top.vhd:L1620-1659` (Vivado trunk: `CLOCK_250MHz` from `CLKFX`, `TDC_CLOCK_PH0/90/180/270` from DCM2)
- **Fine interpolation:** 4 vernier chains (0°, 90°, 180°, 270° phases), each 64 taps × 50 ps/tap (nominal per TAC.docx; VHDL targets 100 ps MAXDELAY, actual ~70 ps/tap measured) ✅ verified 2026-04-09 — TAC.docx §Calculation ("multiplying by 50ps"), `tdc_unit_cont.vhd:L115-116` (MAXDELAY=100ps), L160 ("~70ps/step")
- **Output:** 15-word event packet, sent alongside trigger data

---

## Architecture

### Clock Generation

The main 50 MHz trigger clock is multiplied to **250 MHz** inside the FPGA (DCM CLKFX, ×5) ✅ verified 2026-04-08 — `top.vhd:L1620` (`CLKFX_MULTIPLY=>5, CLKIN_PERIOD=>20.0`) and distributed in **four phases** (PH0/PH90/PH180/PH270), each 90° apart (nominally 1 ns apart) ✅ verified 2026-04-08 — `top.vhd:L829-832, L1543-1585` (`TDC_CLOCK_PH0/90/180/270` BUFGs). Measured chain delays in the Feb 2016 firmware:

| Chain | Phase | Delay to vernier[0] |
|-------|-------|---------------------|
| A | 0° | 1.670 ns |
| B | 90° | 1.826 ns |
| C | 180° | 1.750 ns |
| D | 270° | 1.742 ns |

Phase skew = 156 ps. ✅ verified 2026-04-11 — TAC.docx §Chain delays: "skew of 156ps between chains: 1.670ns...1.826ns...1.750ns...1.742ns" measured via FPGA Editor tool (compilation 20160224). Note: may differ in Vivado port. Temperature-dependent variation expected in the tap delays.

### Vernier (Delay Line) TDC

Each chain uses FPGA carry-chain elements as a **64-tap delay line**. ✅ verified 2026-04-07 — `tdc_unit_cont.vhd:L50` (`DELAY_CHAIN_ODD, DELAY_CHAIN_EVEN: std_logic_vector(63 downto 0)`). When the NIM input fires, the 64-bit thermometer code is latched into flip-flops. A **data compression pipeline** converts the 64-bit code into a **6-bit position value** (0–63) indicating how far the edge propagated. ✅ verified 2026-04-07 — `vernier_pos_finder.vhd:L40` (`TDC_POS: out std_logic_vector(5 downto 0)`). Per-slice `pos_finder.vhd` outputs 4-bit partial results, combined across 9 pipeline stages. Time per tap: nominally **70–50 ps** (VHDL MAXDELAY attribute targets 100 ps; comment in `tdc_unit_cont.vhd:L160` notes ~70 ps/step expected).

### Coarse Counter

A **16-bit counter** at 250 MHz runs in parallel with each vernier chain. It rolls over every **262.144 µs** (65536 × 4 ns). The coarse count is latched with the vernier data and synchronized to the main 48-bit timestamp via Imperative Sync.

### Data Collection State Machines

Three cooperating state machines coordinate data capture:
1. **FIFO READER** (×4, 100 MHz) — monitors each vernier FIFO, reads when data available
2. **TRIG_MON_COLLECT** — watches trigger algorithms; when selected algorithm fires, asserts `WANT_NEXT_TDC` and collects trigger message
3. **TDC_AUTOSAMPLE** — collects TDC data from all four FIFO READERs after `WANT_NEXT_TDC`

A variable delay elapses between `WANT_NEXT_TDC` and when `TDC_AUTOSAMPLE` finishes collection. The **pipeline delay is 350 ns** — any chain with differential > 350 ns vs TDCtsLo is stale and invalid.

---

## Event Data Format

Each TAC-II event is **15 words of 16-bit values** (sent as lower 16 bits of 32-bit VME transactions):

| Word | Name | Description |
|------|------|-------------|
| 1 | Header | Fixed `0xAAAA` (event boundary marker) |
| 2 | trigtyp | Trigger type: upper 8 bits = trigger ID, lower 8 bits = distribution mask |
| 3 | TS high | Bits [47:32] of 48-bit trigger timestamp |
| 4 | TS mid | Bits [31:16] of timestamp |
| 5 | TS low | Bits [15:0] of timestamp |
| 6 | Wheel | 10-bit target wheel encoder position at trigger time |
| 7 | Aux dat | User-defined (reserved, tied to a register) |
| 8 | TDCtsLo | Lower 16 bits of 100 MHz timestamp when TDC data was collected (for validity checking) |
| 9 | trigacks | Bitmap: which other trigger algorithms fired between WANT_NEXT_TDC and TDC collection |
| 10 | Offset0 | 16-bit 250 MHz coarse counter for phase 0° |
| 11 | Offset1 | 16-bit 250 MHz coarse counter for phase 90° |
| 12 | Offset2 | 16-bit 250 MHz coarse counter for phase 180° |
| 13 | Offset3 | 16-bit 250 MHz coarse counter for phase 270° |
| 14 | Val/P1/P0 | Bits [15:12]=valid flags, [11:6]=vernier P0 (0°), [5:0]=vernier P1 (90°) |
| 15 | P3/P2 | Bits [11:6]=vernier P3 (270°), [5:0]=vernier P2 (180°) |

**Note:** This is the raw VME packet. The `tcpReceiverMT` repacks it into a 10-word DIG-compatible format before saving — see `data_structures.md` for the repacked format.

---

## Time Calculation

### Per-Phase Formula

For each valid phase:

```
Time = (Offset × 4 ns) + (per-phase adjustment) - (0.05 ns × vernier_position)   ✅ verified 2026-04-09 — TAC.docx §Calculation ("multiplying that by 50ps (0.050ns)")
```

**Per-phase adjustments** (empirically measured):

| Phase | Offset word | Adjustment |
|-------|-------------|------------|
| 0° | Offset0 | +3 ns |
| 90° | Offset1 | +0 ns |
| 180° | Offset2 | +1 ns |
| 270° | Offset3 | +2 ns |

**Best time = average of all valid phases.**

### Coarse Counter Rollover

The 16-bit coarse counter rolls over every **262.144 µs** (65536 × 4 ns). The full 48-bit timestamp in words 3–5 allows resolving the integer number of rollovers for event-to-event timing calculations.

### Checking Chain Validity

The valid flags in word 14 bits [15:12] are not 100% reliable. Cross-check method:
1. Multiply TDCtsLo × 10 ns → collection time
2. Multiply each Offset × 4 ns → latch time per phase
3. Differential = collection time - latch time
4. Valid chain: differential ≤ 350 ns (the pipeline delay) and > 0

### Example Calculation

From `TAC.docx` example event:
- Offset0 = 0x6262 = 25186 → Time = 25186×4 + 3 - (18×0.05) = **100,746.10 ns**
- Offset1 = 0x6263 = 25187 → Time = 25187×4 + 0 - (22×0.05) = **100,746.90 ns** 
- Offset3 = 0x6262 = 25186 → Time = 25186×4 + 2 - (23×0.05) = **100,744.85 ns**
- Average: **~100,745.9 ns**, std dev ~0.79 ns

---

## Lab Notes & Testing Records

| File | Date | Contents |
|------|------|----------|
| `LabNotes/20210314_TDC.ods` | 2021-03-14 | TDC test data with sync |
| `LabNotes/20210314_TDC_nosync.ods` | 2021-03-14 | TDC test data without sync |
| `LabNotes/tdc.jpg` | — | Oscilloscope capture |
| `LabNotes/Jitter Analysis/` | — | 15 oscilloscope TIF captures (D000–D014) + 10 BMP screenshots + `Jitter Analysis.xls` |
| `LabNotes/DGS/` | — | 5 DGS-specific scope captures (capture0–4) |
| `LabNotes/20210831_lab_notes.odt` | 2021-08-31 | JTA firmware testing notes: sum validation, SZ algorithm verification with digitizer tester |
| `LabNotes/System Timing/mt_tac_20150608.xls` | 2015-06-08 | TAC timing measurements |
| `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/TAC.docx` | 2016-02-24 | Full TAC-II spec (JTA) |
| `DGS_docs/DGS_System_Documentation/System/TAC2.DS4` | — | TAC-II schematic (DesignSpark format) |
| `DGS_docs/DGS_System_Documentation/Modules/TAC.pdf` | — | TAC module PDF spec |

---

## TAC-II in GEBSort (gebsort workflow)

The TAC-II data is sorted by GEBSort using `bin_tac2` (enabled in `GEBSort.chat`):
```
bin_tac2
```
This processes GEB type 15 packets (after `tcpReceiverMT` repacking) to produce TAC timing spectra and coincidence matrices alongside the Ge energy data.

---

## Related

- `data_structures.md` — TAC2 packet format (16-word VME → 10-word repacked)
- `deep_fpga_MTRG_MAIN.md` — MTRG firmware including TDC chain implementation
- `ANLDAQ.md` — data flow from IOC to tcpReceiver
- `dgs_analysis.md` — GEBSort and parquet_pysort analysis

---

*Created: 2026-04-07. Source: TAC.docx (J.T. Anderson, 2016-02-24) + FPGA VHDL.* ✅ verified 2026-04-07 — TAC.docx (DGS_System_Documentation/Firmware/Master_Trigger/TAC.docx): "The total pipeline delay is 350ns, so any differential greater than that, or a negative number, is an old TDC measurement and should not be used."

## Cross-References

- `dgs/deep_fpga_MTRG_MAIN.md` — MTRG Main FPGA: TAC-II TDC integration; NIM IN 2 as TDC stop input
- `dgs/fpga.md` — FPGA firmware overview: TAC-II role in the trigger system
- `dgs/ttcl.md` — TTCL: Frame 16 carries TAC-II TDC data back to DIGs
- `dgs/data_structures.md` — TAC-II TDC data format in GEB binary stream (type 15 = DGSTRIG)
- `dgs/guceiver.md` — Guceiver TAC-II tab: live display of TDC values from TCP stream
- `dgs/connectors.md` — MTRG connector pinouts: NIM IN 2 = TAC-II TDC stop input
