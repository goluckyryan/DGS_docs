# MTRG: tdc_chain_cont.vhd — TDC Chain Controller

**Source:** `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/tdc_chain_cont.vhd` (1036 lines)  
**Entity:** `tdc_chain_cont`  
Stability: C3 - Structural / stable  
**Last analyzed:** 2026-04-24

---

## Table of Contents

1. [Purpose](#purpose)
2. [Port Interface](#port-interface)
3. [Architecture Overview](#architecture-overview)
4. [Sub-components Instantiated](#sub-components-instantiated)
5. [Clock Domains](#clock-domains)
6. [Reset Logic](#reset-logic)
7. [4-Phase TDC Carry Chains (tdc_unit_cont × 4)](#4-phase-tdc-carry-chains)
8. [Fine Counters](#fine-counters)
9. [Trigger ACK Resampling & Accumulation](#trigger-ack-resampling--accumulation)
10. [TDC Request Latch (WANT_NEXT_TDC)](#tdc-request-latch)
11. [TDC Autosample FSM](#tdc-autosample-fsm)
12. [Output FIFO & TDC_AUTOREAD FSM](#output-fifo--tdc_autoread-fsm)
13. [Output Word Format (8-word TDC Event Packet)](#output-word-format)
14. [TDC_FIFO_DATA_READY Signal](#tdc_fifo_data_ready-signal)
15. [Chipscope / ILA Debug](#chipscope--ila-debug)
16. [See Also](#see-also)

---

## Purpose

`tdc_chain_cont` is the top-level controller for the MTRG's Time-to-Digital Converter (TDC) subsystem. It:

1. Instantiates **four parallel carry-chain TDC units** (`tdc_unit_cont`), each clocked by one phase of the 250 MHz quadrature clock (0°, 90°, 180°, 270°), all measuring the same input signal `BIT_IN`.
2. Feeds the four raw 64-bit vernier chains into the **`data_compressor`**, which extracts position and fine-count results via BRAM lookup.
3. Provides a **trigger-driven autosample FSM** that packages TDC results into an 80-bit-wide × 16-deep FIFO as 8-word event packets.
4. Reads those packets back out as 20-bit words, forwards them to `trig_mon_collect` via `TDC_FIFO_WE` / `TDC_FIFO_DATA`, and pulses `TDC_FIFO_DATA_READY` on packet completion.

---

## Port Interface

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `TDC_CLOCK_0/90/180/270` | in | 1 | 250 MHz quadrature sampling clocks |
| `CLOCK_50MHz` | in | 1 | Added 2025-01-31; alternate reset domain |
| `CLOCK_100MHz` | in | 1 | Main control clock (autosample FSM, compressor read, resampling) |
| `TIMESTAMP` | in | 16 | Lower 16 bits of global timestamp |
| `TS_SAMP_PHASE` | in | 1 | Phase flag controlling when timestamp is latched |
| `BIT_IN` | in | 1 | Signal to measure (same input to all 4 TDC chains) |
| `TDC_RESET` | in | 1 | Master reset (resampled into clock domains internally; IMP_SYNC_FLAG) |
| `FAST_TDC_ILA_CTL` | in | 2 | Selects fast ILA mux source |
| `ENABLED_NONVETOED_TRIG_ACK` | in | 8 | Trigger acknowledged flags from 8 algorithms (50 MHz domain) |
| `TDC_TRIG_SEL_MASK` | in | 8 | Selects which trigger ACKs may arm the TDC |
| `ABORT_TDC_HIT` | in | 1 | Asserted externally when TDC measurement takes too long |
| `DIAG_ALLOWED_LATENCY` | in | 8 | Diagnostic: latency allowance (Chipscope only) |
| `DIAG_TDC_MON_STATE` | in | 4 | Diagnostic: external state tracker (Chipscope) |
| `NUM_TDC_WORDS` | in | 6 (9:4) | Number of 20-bit output words per TDC event (normally 32 = 8 × 80-bit writes) |
| `BUF_WANT_NEXT_TDC` | out | 1 | Buffered (flopped) version of `WANT_NEXT_TDC` for external observation |
| `TDC_FIFO_WE` | out | 1 | Write-enable to Mon FIFO 7 (in `trig_mon_collect`) |
| `TDC_FIFO_DATA_READY` | out | 1 | Pulse on completion of one full TDC event packet |
| `TDC_FIFO_DATA` | out | 16 | Data to Mon FIFO 7 |

---

## Architecture Overview

```
BIT_IN ──┬──► TDC_A (tdc_unit_cont, 0°)  ──► TDC_VERNIER(0) ─┐
         ├──► TDC_B (tdc_unit_cont, 90°) ──► TDC_VERNIER(1) ─┤
         ├──► TDC_C (tdc_unit_cont, 180°)──► TDC_VERNIER(2) ─┤
         └──► TDC_D (tdc_unit_cont, 270°)──► TDC_VERNIER(3) ─┤
                                                               │
              FINE_COUNT_BLOCK (×4, per-phase 16-bit free cnt) ┤
                                                               ▼
                                              data_compressor (TDC_COMPRESS)
                                                 │ TDC_VALID(3:0)
                                                 │ TDC_POS(3:0)   [6-bit each]
                                                 │ TDC_OFFSET(3:0)[16-bit each]
                                                 ▼
                                         TDC_AUTOSAMPLE FSM (5-state)
                                                 │ WIDE_FIFO_WE / WIDE_FIFO_DATA
                                                 ▼
                                    FIFO_FWFT_80X16IN_20X64OUT_SEPCLK
                                                 │ WIDE_FIFO_RE / TDC_DATA_OUT
                                                 ▼
                                         TDC_AUTOREAD FSM (2-state)
                                                 │
                                         TDC_FIFO_WE / TDC_FIFO_DATA (→ Mon FIFO 7)
                                         TDC_FIFO_DATA_READY
```

---

## Sub-components Instantiated

| Instance | Entity | Description |
|----------|--------|-------------|
| `TDC_A` | `tdc_unit_cont` | Phase 0° carry-chain TDC; RLOC X0Y0 |
| `TDC_B` | `tdc_unit_cont` | Phase 90° carry-chain TDC; RLOC X2Y0 |
| `TDC_C` | `tdc_unit_cont` | Phase 180° carry-chain TDC; RLOC X4Y0 |
| `TDC_D` | `tdc_unit_cont` | Phase 270° carry-chain TDC; RLOC X6Y0 |
| `TDC_COMPRESS` | `data_compressor` | Converts 4× raw 64-bit verniers → position + fine count |
| `TDC_FIFO` | `FIFO_FWFT_80X16IN_20X64OUT_SEPCLK` | Shallow FIFO: 80-bit in, 20-bit out, separate R/W clocks |
| `TDC_ILA1` | `ila_128` | 128-bit Chipscope ILA (conditional on `TDC_ILA_ENABLE=1`) |
| `TDC_ILA2` | `ila_35` | 35-bit fast ILA in 250 MHz domain (conditional) |
| `TDC_ICON` | `icon2` | Chipscope controller for ILA1+ILA2 |

---

## Clock Domains

| Domain | Frequency | Used for |
|--------|-----------|----------|
| `TDC_CLOCK_0` (0°) | 250 MHz | TDC_A measurement; fine counter A; reset propagation pipeline |
| `TDC_CLOCK_90` (90°) | 250 MHz | TDC_B measurement; fine counter B; reset resample from phase 0 |
| `TDC_CLOCK_180` (180°) | 250 MHz | TDC_C measurement; fine counter C; reset resample from phase 90 |
| `TDC_CLOCK_270` (270°) | 250 MHz | TDC_D measurement; fine counter D; reset resample from phase 180 |
| `CLOCK_100MHz` | 100 MHz | TDC autosample FSM; trigger ACK resampling; FIFO write/read; timestamp latch |

---

## Reset Logic

The TDC reset path is carefully staged across clock domains to prevent metastability:

**At 250 MHz (phase 0):**
```
TDC_RESET → TDC_RESET_pipe (4-stage shift) → RETIMED_PIPELINE_RESET (single pulse on "0011" pattern)
```
`RETIMED_PIPELINE_RESET` produces a single-clock-wide reset pulse when `TDC_RESET` transitions low→high. ✅ verified 2026-04-24 — tdc_chain_cont.vhd:L333-346 (4-stage shift `TDC_RESET_pipe`; case "0011"→RETIMED_PIPELINE_RESET='1', all others='0')

**Phase-to-phase propagation:**
```
SAMP_RESET(0) ← RETIMED_PIPELINE_RESET  (clocked on TDC_CLOCK_0)
SAMP_RESET(1) ← SAMP_RESET(0)           (clocked on TDC_CLOCK_90)
SAMP_RESET(2) ← SAMP_RESET(1)           (clocked on TDC_CLOCK_180)
SAMP_RESET(3) ← SAMP_RESET(2)           (clocked on TDC_CLOCK_270)
```
Each `SAMP_RESET(n)` resets its corresponding TDC unit and fine counter. ✅ verified 2026-04-24 — tdc_chain_cont.vhd:L356 (`SAMP_RESET(0)<=RETIMED_PIPELINE_RESET`), L364 (`SAMP_RESET(1)<=SAMP_RESET(0)` on TDC_CLOCK_90), L371/L378 (phases 180/270 similarly)

**At 100 MHz (`RESET_100`):**
```
TDC_RESET → RESET_PIPELINE (4-stage) → RESET_100 (single pulse on "0011" pattern)
```
`RESET_100` resets the autosample FSM, FIFO, trigger ACK accumulators, and data compressor. ✅ verified 2026-04-24 — tdc_chain_cont.vhd:L388-393 (`RESET_PIPELINE` 4-stage shift on CLOCK_100MHz; `if RESET_PIPELINE="0011" then RESET_100<='1'`)

---

## 4-Phase TDC Carry Chains

Four `tdc_unit_cont` instances are placed at fixed RLOC locations 2 columns apart (X0Y0, X2Y0, X4Y0, X6Y0), ensuring spatial separation of the four phases in the Xilinx carry chain fabric. ✅ verified 2026-04-24 — tdc_chain_cont.vhd:L285-291 (`attribute RLOC of TDC_A/B/C/D :label is "X0Y0"/"X2Y0"/"X4Y0"/"X6Y0"`)

Each unit:
- Samples `BIT_IN` using its phase-specific 250 MHz clock
- Propagates signal through a 64-element carry chain (MUXCY + XORCY primitives)
- Outputs a 64-bit thermometer pattern (`TDC_VERNIER_OUT`) and a run-monitor bit (`TDC_RUN_MON`)

The `RUN_MON_BLOCK` generate loop flops each `TDC_RUN_MON` through a pipeline synchronized to its respective phase clock; `TDC_RUN_MON_pipe` is reset by `SAMP_RESET`.

---

## Fine Counters

**`FINE_COUNT_BLOCK`** (generate loop, 0 to 3):
- Each phase has a 16-bit free-running counter clocked by its respective `TDC_CLOCK_x`
- Reset to 0x0000 when `SAMP_RESET(n) = '1'`
- Counts indefinitely once reset releases
- Provides a coarse timing reference ("fine count" = number of 4 ns TDC clocks from reset to hit)

All four `TDC_FINE_COUNT(n)` vectors feed `data_compressor` alongside the raw vernier chains.

---

## Trigger ACK Resampling & Accumulation

`ENABLED_NONVETOED_TRIG_ACK` arrives from the 50 MHz `mclk` domain. Domain crossing uses a **toggle-phase resampler**:

**`RESAMPLE_BLOCK`** (generate loop, 1 to 8):
- `SAMPLE_PHASE(i)` alternates '1'/'0' every 100 MHz clock
- On `SAMPLE_PHASE='1'` half-cycles: `SAMPLED_TRIG_ACK(i) ← ENABLED_NONVETOED_TRIG_ACK(i)` (if `TDC_TRIG_SEL_MASK(i)='1'`)
- On `SAMPLE_PHASE='1'` half-cycles: `ALL_SAMPLED_TRIG_ACK(i) ← ENABLED_NONVETOED_TRIG_ACK(i)` (all triggers, no mask)
- On `SAMPLE_PHASE='0'` half-cycles: both forced to '0'
- Effect: effectively shortens any 50 MHz pulse to at most one 100 MHz tick, avoiding double-counting

**`ACCUMULATED_TRIG_ACKs`** (OR accumulator in 100 MHz domain):
- Sets bits whenever any `SAMPLED_TRIG_ACK` is non-zero (i.e., masked trigger fired)
- Also ORs in `ALL_SAMPLED_TRIG_ACK` to track all concurrent triggers
- Cleared on reset, abort, or on ACK_OF_VALID (when data is captured)
- Recorded in the TDC output packet (word 2: trigger bitmap)

---

## TDC Request Latch

**`TDC_REQUEST_PROC`** (100 MHz):
- `WANT_NEXT_TDC` latches to '1' when a masked trigger fires (`SAMPLED_TRIG_ACK ≠ 0`)
- Clears on reset, `ABORT_TDC_HIT`, or when `ACK_OF_VALID ≠ 0`
- Stays latched until the data compressor acknowledges it has captured a hit post-trigger
- Propagates to `data_compressor` to arm it for the next valid TDC event

**Timestamp capture** (`TIMESTAMP_RESAMPLE`):
- `RESAMPLED_TIMESTAMP` latches `TIMESTAMP` when `SAMPLE_PHASE(1) = TS_SAMP_PHASE` and the FSM is in `RELATCH` or `WAIT_TDC` state

---

## TDC Autosample FSM

**States:** `IDLE`, `WAIT_TDC`, `RELATCH`, `WRITE_DATA`, `WRITE_DATA2` ✅ verified 2026-04-24 — tdc_chain_cont.vhd:L130 (`type TDC_AUTOSAMPLE_STATES is (IDLE, WAIT_TDC, RELATCH, WRITE_DATA, WRITE_DATA2)`)

| State | Action |
|-------|--------|
| `IDLE` | Clears FIFO write signals; transitions immediately to `WAIT_TDC` unless reset |
| `WAIT_TDC` | Polls `TDC_VALID`; on reset/abort → IDLE; when `TDC_VALID ≠ 0` → latches `COMPRESSOR_DATA[0..5]` → RELATCH |
| `RELATCH` | Double-samples `COMPRESSOR_DATA` (guards against late arrivals in adjacent compressor channels); asserts `ACK_OF_VALID ← TDC_VALID`; → WRITE_DATA |
| `WRITE_DATA` | Writes first 80-bit word to `WIDE_FIFO`: [timestamp ‖ trig_ack_bitmap ‖ OFFSET_A ‖ OFFSET_B]; → WRITE_DATA2 |
| `WRITE_DATA2` | Writes second 80-bit word: [OFFSET_C ‖ OFFSET_D ‖ POS_AB ‖ POS_CD]; → WAIT_TDC |

**COMPRESSOR_DATA packing** (per TDC_VALID bit pattern):
- `COMPRESSOR_DATA(0)` = `TDC_OFFSET(0)` (phase A fine count, or 0 if A invalid)
- `COMPRESSOR_DATA(1)` = `TDC_OFFSET(1)` (phase B fine count, or 0 if B invalid)
- `COMPRESSOR_DATA(2)` = `TDC_OFFSET(2)` (phase C fine count, or 0 if C invalid)
- `COMPRESSOR_DATA(3)` = `TDC_OFFSET(3)` (phase D fine count, or 0 if D invalid)
- `COMPRESSOR_DATA(4)` = `TDC_VALID[3:0]` & `TDC_POS(0)[5:0]` & `TDC_POS(1)[5:0]`
- `COMPRESSOR_DATA(5)` = `"0000"` & `TDC_POS(2)[5:0]` & `TDC_POS(3)[5:0]`

---

## Output FIFO & TDC_AUTOREAD FSM

**FIFO:** `FIFO_FWFT_80X16IN_20X64OUT_SEPCLK`
- Write side: 80-bit wide, written 2× per TDC event (WRITE_DATA + WRITE_DATA2)
- Read side: 20-bit wide FWFT; 4:1 serialization means each 80-bit write produces 4 × 20-bit reads
- 2 writes × 4 reads = **8 × 20-bit reads** per event = 8 × `TDC_FIFO_WE` pulses

**`TDC_AUTOREAD` FSM** (2-state, IDLE / READ_FIFO):
- IDLE: waits for `WIDE_FIFO_EMPTY = '0'`; loads `WIDE_RE_COUNT ← NUM_TDC_WORDS`; asserts `WIDE_FIFO_RE`
- READ_FIFO: decrements `WIDE_RE_COUNT` each cycle until zero; deasserts `WIDE_FIFO_RE`

**`TDC_DATA_OUT_LATCH`:**
- `TDC_FIFO_WE ← WIDE_FIFO_RE` (1-clock delay)
- `TDC_FIFO_DATA ← TDC_DATA_OUT[15:0]` (lower 16 of 20-bit output)
- `TDC_FIFO_DATA_READY` pulses for 1 cycle on the falling edge of `WIDE_FIFO_RE_pipe`
  - Unless `ABORT_TDC_HIT` is asserted (forces `int_TDC_FIFO_DATA_READY = '0'`)

---

## Output Word Format

Each TDC event is written as **two 80-bit FIFO entries**, repacked into **eight 20-bit output words** (upper 4 bits are framing tags, lower 16 bits are data):

✅ verified 2026-04-24 — tdc_chain_cont.vhd:L820 (WRITE_DATA 80-bit word) and L834 (WRITE_DATA2 80-bit word) confirm all 8 framing tags and data assignments exactly.

| Word | Framing Tag (bits 19:16) | Data (bits 15:0) | Content |
|------|--------------------------|-------------------|---------|
| 1 | `0xF` (1111) | `RESAMPLED_TIMESTAMP` | Lower 16 bits of event timestamp |
| 2 | `0xE` + `111000000000` | `ACCUMULATED_TRIG_ACKs[7:0]` | Bitmap of all trigger algorithms that fired |
| 3 | `0xD` (1101) | `COMPRESSOR_DATA(0)` | TDC fine count, phase A (0 if invalid) |
| 4 | `0xC` (1100) | `COMPRESSOR_DATA(1)` | TDC fine count, phase B (0 if invalid) |
| 5 | `0xB` (1011) | `COMPRESSOR_DATA(2)` | TDC fine count, phase C (0 if invalid) |
| 6 | `0xA` (1010) | `COMPRESSOR_DATA(3)` | TDC fine count, phase D (0 if invalid) |
| 7 | `0x9` (1001) | `TDC_VALID[3:0]` & `POS_A[5:0]` & `POS_B[5:0]` | Validity flags + vernier positions A & B |
| 8 | `0x8` (1000) | `0000` & `POS_C[5:0]` & `POS_D[5:0]` | Vernier positions C & D |

Fine count interpretation: each LSB = 1 TDC clock period = 4 ns (at 250 MHz).  
Vernier position: 0–63, representing sub-clock-cycle TDC edge position within carry chain.

---

## TDC_FIFO_DATA_READY Signal

`TDC_FIFO_DATA_READY` (internally `int_TDC_FIFO_DATA_READY`) is a 1-cycle pulse that fires after all 20-bit output words for one TDC event have been written to Mon FIFO 7. It signals upstream logic (in `trig_mon_collect` / `registers.vhd`) that a complete TDC record is available.

Force-cleared by `ABORT_TDC_HIT` regardless of FIFO state.

---

## Chipscope / ILA Debug

Two ILA blocks, enabled via generic `TDC_ILA_ENABLE = 1`:

**`TDC_ILA1`** (128-bit, 100 MHz):
- Captures autosample FSM state, ACK signals, FIFO status, TDC_VALID, all 4 POS + OFFSET values, trigger ACK bitmaps, RESET_100, SAMPLE_PHASE, ABORT_TDC_HIT, DATA_READY

**`TDC_ILA2`** (35-bit, 250 MHz phase 0):
- Captures: DIAG_SLICER[15:0], WANT_NEXT_TDC, TDC_VALID(0), FAST_ANY_TRIG
- Upper 16 bits selected by `FAST_TDC_ILA_CTL`:
  - `00` → TDC_FINE_COUNT(0)
  - `01` → DIAG_RAW0[31:16]
  - `10` → DIAG2[15:0]
  - `11` → TDC_POS(0) & TDC_VALID(0) & '0' & TDC_OFFSET(0)[7:0]

`FAST_ANY_TRIG`: single-bit 250 MHz resample of `ENABLED_NONVETOED_TRIG_ACK ≠ 0`, for fast ILA triggering.

---

## See Also

- [MTRG_support_modules.md](MTRG_support_modules.md) — `tdc_unit_cont` (jta_odelay.vhd, 64-element carry chain), `data_compressor`, `vernier_pos_finder`
- [MTRG_pos_finder.md](MTRG_pos_finder.md) — `pos_finder.vhd` (thermometer→position ROM lookup)
- [MTRG_registers.md](MTRG_registers.md) — VME register map; `NUM_TDC_WORDS`, `TDC_TRIG_SEL_MASK`, `ABORT_TDC_HIT` sources; Mon FIFO 7
- [MTRG_comp_defs.md](MTRG_comp_defs.md) — `tdc_chain_cont` component declaration with port list
- [deep_fpga_MTRG_MAIN.md](../deep_fpga_MTRG_MAIN.md) — MTRG top-level integration
- [fpga.md](../fpga.md) — VHDL index
