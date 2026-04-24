# MTRG Support Modules — VHDL Reference

Stability: C3 - Structural / stable

_Source: `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/` — code-read 2026-04-24_

These modules are support infrastructure used throughout the MTRG firmware but not previously documented in a dedicated file. They are referenced from `MTRG_top.md`, `MTRG_mstr_mach.md`, `MTRG_trig_algo_support.md`, and other MTRG notes.

---

## Table of Contents

1. [timestamp.vhd — 48-bit Global Counter](#timestampvhd--48-bit-global-counter)
2. [data_compressor.vhd — TDC Vernier Position Extractor](#data_compressorvhd--tdc-vernier-position-extractor)
3. [link_tx_block.vhd — SERDES Link Output / DC Balance Fan-out](#link_tx_blockvhd--serdes-link-output--dc-balance-fan-out)
4. [remote_trig_support.vhd — Cross-System (Link R) Trigger Algorithm](#remote_trig_supportvhd--cross-system-link-r-trigger-algorithm)
5. [trig_mon_collect.vhd — Trigger Monitor FIFO Collector](#trig_mon_collectvhd--trigger-monitor-fifo-collector)
6. [trigger_data_types.vhd — VHDL Type Definitions Package](#trigger_data_typesvhd--vhdl-type-definitions-package)
7. [cpld_trig.vhd — CPLD Fast-Sum Trigger Algorithm Wrapper](#cpld_trigvhd--cpld-fast-sum-trigger-algorithm-wrapper)
8. [jta_odelay.vhd / tdc_unit_cont — Continuously-Running TDC Vernier Chain](#jta_odelayvhd--tdc_unit_cont--continuously-running-tdc-vernier-chain)
9. [jta_vernier_pos_finder.vhd / vernier_pos_finder — TDC Position Decoder](#jta_vernier_pos_findervhd--vernier_pos_finder--tdc-position-decoder)

---

## timestamp.vhd — 48-bit Global Counter

**Author:** John Anderson  
**Purpose:** Provides the MTRG with a 48-bit free-running timestamp counter, compatible with the DGS digitizer timestamp, plus a set of rate-selectable test-pulse edge flags.

### Clock Relationship

The timestamp runs at **50 MHz** (the MTRG master clock `mclk`) but **increments by 2** each clock cycle, giving an effective least-count of **20 ns** — matching the 100 MHz nominally used in digitizers. Bit 1 therefore toggles at 50 MHz, not bit 0 (which is always 0 and effectively unused).

### Bit Period Reference Table (selected)

> **Period convention:** Values are *toggle periods* — the interval between successive bit transitions (= 2^(n−1) × 20 ns for bit n). The in-code TEST RATE comments use full-cycle period (2× toggle), which was the source of earlier KB errors.

| Bit | Toggle Period | Toggle Frequency |
|-----|--------------|-----------------|
| 1 | 20 ns | 50 MHz |
| 8 | 2.56 µs | 390.6 kHz |
| 10 | 10.24 µs | 97.66 kHz |
| 15 | 327.7 µs | 3.052 kHz |
| 19 | 5.243 ms | 190.7 Hz |
| 26 | 671.1 ms | 1.490 Hz |
| 43 | ~1.018 days | — |
| 47 | ~16.29 days | — |

Full rollover at 48 bits ≈ 32.6 days.

✅ verified 2026-04-24 — timestamp.vhd header bit-period table (L17–L68). Prior KB entry had bit-19 = 10.49 ms and bit-26 = 1.34 s; both were off by 2× (full-cycle vs. toggle-period confusion).

### Synchronization Logic

On reset, the counter loads `STARTING_TIMESTAMP + 14` (a fixed latency offset). ✅ verified 2026-04-24 — timestamp.vhd:L120 (`xTIMESTAMP <= STARTING_TIMESTAMP + 14`). During normal operation it increments freely by 2 each clock. ✅ verified 2026-04-24 — timestamp.vhd:L128,L133 (`xTIMESTAMP <= xTIMESTAMP + 2`).

If `LINK_L_SERDES_SM_LOCKED` and `REMOTE_TIMESTAMP_ENABLE` are both asserted, the MTRG acts as a **timestamp slave** (like a digitizer): on each `REMOTE_SYNC` pulse it loads `REMOTE_TIMESTAMP + STARTING_TIMESTAMP` rather than free-running. ✅ verified 2026-04-24 — timestamp.vhd:L122-L125 (modified 20150910 by JTA). This allows an external master to keep multiple MTRG units in sync.

### Test Rate Edge Flags (`TS_EDGE_FLAGS[6:0]`)

Seven selectable test-pulse rates derived from timestamp bit transitions. Polarity note: edge flags fire on the **1→0** transition (i.e., at the 0x3F→0x40 rollover boundary), matching the digitizer tester convention (modified 2022-08-20).

| Index | TS Bit | Toggle Period | Approx. Rate |
|-------|--------|--------------|--------------|
| 0 | 26 | 671 ms | TEST RATE 0 |
| 1 | 23 | 83.9 ms | TEST RATE 1 |
| 2 | 21 | 21.0 ms | TEST RATE 2 |
| 3 | 19 | 5.24 ms | TEST RATE 3 |
| 4 | 15 | 327.7 µs | TEST RATE 4 |
| 5 | 10 | 10.24 µs | TEST RATE 5 (emulates DGS target rate) |
| 6 | 8 | 2.56 µs | TEST RATE 6 (pileup test) |

✅ verified 2026-04-24 — TS_IDX constant (timestamp.vhd:L101) and header bit-period table. Periods are toggle periods (2^(n−1) × 20 ns). In-code TEST RATE comments use full-cycle period; do not rely on them for quantitative values.

`TS_EDGE_FLAGS(7)` is hardwired to `'0'` (reserved). ✅ verified 2026-04-24 — timestamp.vhd:L148.

---

## data_compressor.vhd — TDC Vernier Position Extractor

**Authors:** Katelyn Kufahl (original 2013), modified John Anderson (2015, 2025-06)  
**Purpose:** Converts raw 64-bit TDC data from each of four vernier timing chains into a compact 6-bit position + 16-bit fine-count output suitable for timestamp resolution in the MTRG's TAC/TDC system.

### Architecture Overview

The MTRG's timing resolution system uses **four parallel vernier delay chains**, each running in its own clock domain (`clk_tdc[3:0]`). The `data_compressor` instantiates four copies of `vernier_pos_finder`, each in its own clock domain, then bridges the outputs to the 100 MHz main clock domain via cross-domain FIFOs.

```
tdc_data[i] (64-bit) ──┐
tdc_fine_count[i] ──── vernier_pos_finder ─► FIFO_22W16D_FWFT_BRAM ─► FIFO_READER ─► TDC_POS[i], TDC_OFFSET[i]
                         (clk_tdc[i] domain)   (dual-clock, FWFT)       (CLOCK_100MHz domain)
```

### Signal Descriptions

| Signal | Direction | Width | Description |
|--------|-----------|-------|-------------|
| `tdc_data[3:0]` | in | 4×64b | Raw vernier chain bit-pattern per clock phase |
| `tdc_fine_count[3:0]` | in | 4×16b | Coarse counter from delay chain logic |
| `CLOCK_100MHz` | in | 1b | Global domain for FIFO read-side |
| `WANT_NEXT_TDC` | in | 1b | Asserted when the caller wants the next measurement |
| `ACK_OF_VALID[3:0]` | in | 4b | Acknowledgment that `TDC_VALID` has been processed |
| `TDC_VALID[3:0]` | out | 4b | Asserted (in 100 MHz domain) when fresh data is available AND `WANT_NEXT_TDC` was set |
| `ANY_TDC_VALID[3:0]` | out | 4b | Asserted immediately when data is available, regardless of `WANT_NEXT_TDC` |
| `TDC_POS[3:0]` | out | 4×6b | Vernier edge position (0–63) |
| `TDC_OFFSET[3:0]` | out | 4×16b | Fine count (offset / coarse count) |
| `ABORT_TDC_HIT` | in | 1b | Clears WAIT_ACK state if no TDC hit arrives in time |

### FIFO Reader State Machine

Each of the four channels has a two-state FSM:

- **IDLE:** Wait for FIFO non-empty. On data: capture `TDC_POS`/`TDC_OFFSET`, assert `ANY_TDC_VALID`. If `WANT_NEXT_TDC`, also assert `TDC_VALID` and go to WAIT_ACK.
- **WAIT_ACK:** Hold outputs stable. Clear VALID on `ACK_OF_VALID` or `ABORT_TDC_HIT`, return to IDLE.

### FIFO Implementation

Cross-domain FIFOs: `FIFO_22W16D_FWFT_BRAM` (22-bit wide, 16-deep, first-word-fall-through, BRAM-based). Write side uses `clk_tdc[i]`, read side uses `CLOCK_100MHz`. Data packing: `DIN[5:0]` = `TDC_POS`, `DIN[21:6]` = `TDC_OFFSET`.

---

## link_tx_block.vhd — SERDES Link Output / DC Balance Fan-out

**Author:** John Anderson  
**Purpose:** Takes the 16-bit TTCL command word from the master state machine, DC-balances it, and fans it out to all 11 SERDES output links of the MTRG. Each link can be individually masked.

### Link Mapping

| Links | Count | Source | Notes |
|-------|-------|--------|-------|
| 1–8 (A–H) | 8 | `CONTROL_DATA` (TTCL command) | Shared DC-balanced output to Routers |
| 9 (Link L) | 1 | `LINK_L_COMMAND_OUT` | Separate DC balance; used for downlink to secondary MTRG or daisy-chain |
| 10 (Link R) | 1 | `LINK_R_COMMAND_OUT` | Separate DC balance; used for cross-system remote trigger connection |
| 11 (Link U) | 1 | `LINK_U_COMMAND_OUT` | Separate DC balance; used for MγRIAD uplink |

Each link group has its own `dc_balance_mach` instance. The enable bit for each DC balance machine comes from `MISC_CTL2_REG` bits 15, 11, 12, 13 for the main, L, R, U groups respectively. ✅ verified 2026-04-23 — link_tx_block.vhd:L99,L143,L176,L209

### Masking Behavior

`INPUT_LINK_MASK_REG[10:0]` controls per-link masking:

- **Links 1–8 (bits 0–7):** If masked (`INPUT_LINK_MASK_REG(i-1) = '1'`), normally outputs fixed pattern `0x00FF` (SYNC pattern) instead of TTCL data. **Exception:** if `EN_DATA_ALWAYS = '1'`, the DC-balanced TTCL data is sent even on masked links. This was added 2024-03-06 to support Pixie-XL digitizers which require TTCL even on "masked" links. ✅ verified 2026-04-23 — link_tx_block.vhd:L105,L114-L117 (comment: "modified 20240306 by JTA to always send the DC BALANCED data, masked or not")
- **Links L/R (bits 8/9):** If masked, outputs `0xFF00` fixed pattern. ✅ verified 2026-04-23 — link_tx_block.vhd:L155,L188
- **Link U (bit 10):** If masked, outputs `0xFF00` fixed pattern. ✅ verified 2026-04-23 — link_tx_block.vhd:L221

### DC Balance

Data is promoted to 18 bits (`'0' & data16 & '1'`) before passing to `dc_balance_mach`, which ensures the SERDES running disparity stays balanced. The `disp_calc()` function (in `trigger_data_types.vhd`) counts +1/-1 for each bit to compute the signed disparity.

### Monitor MUX

`MON2_FIFO_SEL_REG[3:0]` selects which link's TX data (bits [16:1] of the 18-bit DC-balanced word) appears on `MON2_FIFO_IN` for Chipscope/FIFO monitoring. Values 1–11 select links 1–11; any other value returns `0xDEAD`.

---

## remote_trig_support.vhd — Cross-System (Link R) Trigger Algorithm

**Author:** John Anderson (2013, major additions 2015, 2018, 2024, 2025)  
**Purpose:** Implements the MTRG trigger algorithm for triggers received from a **remote master** via Link R. Supports both timestamp-matched coincidence mode and simple delayed-flag mode, optionally requiring coincidence with other local trigger algorithms.

This is one of the eight algorithm slots in the MTRG (`TRIG_LOGIC6` or `TRIG_LOGIC7` depending on configuration). It uses `trig_algo_support` as its base (same as all other MTRG algorithms) for FIFO management, prescale, holdoff, event counting, and throttle.

### Operating Modes

**Mode 0 — Timestamp retiming (default):**  
The timestamp carried in the Link R trigger message is extracted and compared against the current local MTRG timestamp (plus a programmable offset `REMOTE_TRIGGER_TS_OFFSET` minus `REMOTE_TRIG_DIG_OFFSET`). The trigger fires when the local timestamp equals or exceeds the offset remote timestamp. This strips out Link R transmission jitter, creating a precise timestamp-matched trigger.

The net offset applied is:  
`NET_TS_OFFSET = REMOTE_TRIGGER_TS_OFFSET - REMOTE_TRIG_DIG_OFFSET`

If the local timestamp is already past the target when the comparison starts (rare; indicates misconfigured offsets), the trigger fires immediately and `DIAG_REMOTE_TRIGGER_DELAY_ERROR` is set.

**Mode 1 — Fixed delay (MyRIAD-like):**  
The raw `REMOTE_TRIG_FLAG` is passed through a programmable delay line (`REMOTE_TRIG_DELAY_REG` count of 20 ns ticks). Trigger fires when the delayed copy emerges. The timestamp in the trigger message is the local timestamp at that moment. Used when the remote system has no digitizers and timestamp matching is not meaningful.

### Optional Local Coincidence

`OTHER_TRIGGER_MASK[7:1]` allows the remote trigger to be held until a coincidence with any of the other 7 local MTRG algorithms is confirmed:

| Mask bit | Algorithm |
|----------|-----------|
| 1 | Manual / NIM-in (AUX) / Target Wheel |
| 2 | SumX |
| 3 | SumY |
| 4 | SumXY |
| 5 | CPLD fast sum |
| 6 | GITMO or other Remote Master |
| 7 | Remote Master |

If `OTHER_TRIGGER_MASK = 0`, the remote trigger fires alone. If set, `overlap_mach` is used to confirm a time-window coincidence. The `LOCAL_TRIG_DELAY_REG` applies a programmable delay to the local side (since the local trigger typically solves before the remote message arrives).

### Timestamp FIFO Buffer

Remote trigger timestamps are immediately pushed into a 48×64 FWFT FIFO (`TRIG_MSG_FIFO`) on arrival. The comparison state machine reads them out one at a time, allowing back-to-back remote triggers to queue up without loss.

### Subtype Codes

| Condition | TRIGGER_SUBTYPE |
|-----------|----------------|
| Mode 0, no local coincidence | `0x6` & `'0'` & `REMOTE_TRIG_TYPE[2:0]` |
| Mode 0, with local coincidence | `0x68` |
| Mode 1 (delay mode) | `0x7` & `'0'` & `REMOTE_TRIG_TYPE[2:0]` |

---

## trig_mon_collect.vhd — Trigger Monitor FIFO Collector

**Author:** John Anderson (2015-10-22)  
**Purpose:** Collects trigger monitoring records from all 8 trigger algorithm FIFOs and, optionally, TDC timing data, and merges them into a single output FIFO (Monitor FIFO 7, accessible over VME).

This runs at **100 MHz** (asynchronous to the 50 MHz trigger system) and coordinates two data streams:
1. **TRIG_MON_FIFO:** One per algorithm. Each fills when a trigger occurs, writing a record of trigger type, timestamp, and detector state at trigger time.
2. **TDC FIFO:** One globally. Filled by `tdc_chain_cont` when a TDC measurement completes after a trigger.

### Key Design Points

- `TDC_TRIG_SEL_MASK` selects **exactly one** trigger algorithm to associate with TDC records (enforced at top level to be single-bit). This ensures 1:1 correlation between TDC FIFO and TRIG_MON_FIFO events.
- The **other 7** (non-selected) TRIG_MON_FIFOs are continuously flushed (RE held high) to prevent overflow.
- `TRIGGER_MON_FLAG(i)` pulses when a full event is available in TRIG_MON_FIFO(i). The state machine waits for this flag before attempting to read.
- `TDC_FIFO_DATA_READY` is a pulse indicating TDC data is fully written; a latch inside holds this until the state machine processes it.
- `ABORT_TDC_HIT` is asserted if a TDC event doesn't arrive within a timeout window, allowing recovery.
- `SKIP_TDC_DATA` flag: if set, fake (zero) TDC data is used instead of real TDC FIFO data — useful for testing.
- `WROTE_MON7_EVENT` pulses synchronous with the last write of each merged event to Monitor FIFO 7 (added 2020-06-23).
- `USER_PACKAGE_DATA[9:0]` (10 bits, added 2021-06-15) is inserted between the trigger data and TDC data in the output stream — used for experiment-specific metadata tagging.

### Output Destination

Output goes to **Monitor FIFO 7** (instantiated in the `registers` subdesign), readable over VME. Used for offline analysis of trigger rates and TDC calibration data without perturbing the main data stream.

---

## trigger_data_types.vhd — VHDL Type Definitions Package

**Author:** John Anderson  
**Purpose:** Shared VHDL package defining custom array types and utility functions used throughout the MTRG firmware.

### Named Array Types (JTA prefix = J. T. Anderson)

| Type | Definition | Used for |
|------|-----------|---------|
| `JTA_4X16_Array` | array(4→1) of slv(15:0) | 4-element 16-bit arrays |
| `JTA_8X4_Array` | array(8→1) of slv(3:0) | 8 Router × 4-bit status per channel |
| `JTA_8X8_Array` | array(8→1) of slv(7:0) | 8 Router × 8-bit values |
| `JTA_8X16_Array` | array(8→1) of slv(15:0) | 8 Router × 16-bit values (most common) |
| `JTA_8X18_Array` | array(8→1) of slv(17:0) | 8 Router × 18-bit SERDES raw words |
| `JTA_11X18_Array` | array(11→1) of slv(17:0) | 11 TX links × 18-bit DC-balanced words |
| `JTA_8X24_Array` | array(8→1) of slv(23:0) | 8 Router × 24-bit energy sub-sums |
| `JTA_8X40_Array` | array(8→1) of slv(39:0) | 8 Router × 40-bit hit pattern bitmaps |
| `JTA_8X8X8_Array` | array(8→1) of JTA_8X8_Array | 64-entry 8-bit (8 Masters × 8 Routers) |
| `JTA_8X8X16_Array` | array(8→1) of JTA_8X16_Array | 64-entry 16-bit |
| `JTA_8X8X40_Array` | array(8→1) of JTA_8X40_Array | 64-entry 40-bit |

### Indexed Array Types (from digitizer package)

`array_of_slv_N_0` for N = 1–21, 23, 31, 35, 62, 63, 511 — unconstrained arrays of std_logic_vector(N downto 0), used wherever variable-length arrays of matching widths are needed.

### Enumeration Types

| Type | Values |
|------|--------|
| `tCMD_FORMAT` | `DGS_MASTER`, `DGS_ROUTER`, `GRETINA_MASTER` |
| `tBUILD_TYPE` | `DGS_MASTER`, `DFMA_MASTER` |

### Utility Functions

- **`itoa(n: integer) → string`** — Integer to ASCII string conversion; recursive; handles negative numbers.
- **`disp_calc(input: std_logic_vector) → std_logic_vector(3:0)`** — DC disparity calculator: counts +1 for each '1' bit, -1 for each '0', returns signed half-sum as 4-bit vector. Used by DC balance logic.

---

---

## cpld_trig.vhd — CPLD Fast-Sum Trigger Algorithm Wrapper

**Author:** MBO (Michael Oberling, date implied by file history)  
**Purpose:** Thin wrapper that adapts the CPLD hardware fast-sum trigger signal (`SIMPLE_TRIGGER`) into the standard MTRG trigger-algorithm framework by instantiating `trig_algo_support` with no custom preprocessing.

### Design Pattern

`cpld_trig` has **no behavioral logic of its own**. All trigger logic — FIFO management, prescaling, holdoff, veto, monitor FIFOs, event counting, throttle — is handled by `trig_algo_support` (see `MTRG_trig_algo_support.md`). `cpld_trig` exists solely to:
1. Name the algorithm (CPLD fast-sum) semantically in the design hierarchy.
2. Wire `SIMPLE_TRIGGER → TRIGGER_OCCURRED` and `TRIG_TYPE → TRIGGER_TYPE_CODE`.
3. Pass all generic ports to `trig_algo_support` unchanged.

This mirrors the same pattern used by `local_trig_coinc`, `remote_trig_support`, and the MYRIAD trigger modules — each is a thin wrapper around `trig_algo_support`.

### Signal of Note

| Signal | Direction | Description |
|--------|-----------|-------------|
| `SIMPLE_TRIGGER` | in | Edge from CPLD indicating fast-sum condition met |
| `TRIG_TYPE[7:0]` | in | Trigger type code passed to output packet |
| `TRIGGER_HOLDOFF[15:0]` | in | Self-veto duration in 20 ns ticks (added 2025-10-22) |
| `TRIG_HOLDOFF_ENBL` | in | Enable holdoff logic (added 2025-10-22) |

### Notes

- Comment in `top.vhd` context: "the OR of the various conditions that cause CPLD_TRIG is done in TOP.VHD"
- Trigger monitoring hooks (shadow FIFO with `LOCAL_TRIG_MON_DET_DATA` / `XTRA_DATA`) added October 2015
- Holdoff support added 2025-10-22 — same addition applied across all algorithm wrappers

---

## jta_odelay.vhd / tdc_unit_cont — Continuously-Running TDC Vernier Chain

**Author:** JTA  
**Entity name:** `tdc_unit_cont` (file named `jta_odelay.vhd` — name is historical/misleading; the design is **not** an ODELAY primitive; the file header says "work in progress to make an ODELAY for Virtex-4 that doesn't have one")  
**Purpose:** Implements a **64-element continuously-running Carry-Chain TDC** in Xilinx Virtex-4 FPGA fabric. Measures fine timing of an input signal by propagating it through a series of MUXCY/XORCY primitives placed on the carry chain, then sampling the chain state into flip-flops at 250 MHz.

### Architecture

- **Clock:** 250 MHz TDC sampling clock (`TDC_CLOCK`)
- **Input:** Single-bit signal to be measured (`BIT_IN`)
- **Chain:** 64 stages, alternating ODD/EVEN delay chain nodes
  - Each stage: MUXCY_L (carry propagation) → XORCY_L (pickoff to flip-flop)
  - Two pickoff points per CLB slice → 128 total sample points over 64 CLB columns
  - Nominal step size: ~70 ps/stage → total chain length ~4.5 ns (just over one 250 MHz period)
- **Output:** `TDC_VERNIER_OUT[63:0]` — raw thermometer code from the 64 sampling flip-flops
- **Placement:** RLOC-constrained: slice 0 placed at X0Y0, loop generates X0Y1..X0Y31

### Vernier Pattern

Chain is a **ripple carry** structure:
```
BIT_IN → MUXCY_L(even(0)) → MUXCY_L(odd(0)) → MUXCY_L(even(1)) → ...
              ↓ XORCY_L                ↓ XORCY_L
           FF1d(0)                 FF2d(0)
```
Sampling flip-flops capture the thermometer code. The transition position in the code gives sub-clock-period timing of `BIT_IN`.

### Notes

- Architecture-specific to **Virtex-4** (uses MUXCY_L, XORCY_L, XORCY_D, MUXCY_D, FDCPE primitives)
- `MAXDELAY` constraint on both DELAY_CHAIN_EVEN and DELAY_CHAIN_ODD signals: 100 ps (ensures synthesis preserves timing)
- `TDC_RUN_MON` output: one of the TDC_RUNx flip-flops, used for external monitoring / diagnostics
- The file also references a `BIT_OUT` and `SLOW_CLOCK_SEL` in the final output line — these appear to be leftover from a mux-output mode; not part of the entity port (likely dead code / scratch)
- Instantiated by `data_compressor.vhd` — the `jta_vernier_pos_finder` module then converts the raw thermometer output to a 6-bit position

---

## jta_vernier_pos_finder.vhd / vernier_pos_finder — TDC Position Decoder

**Author:** JTA  
**Date:** 2025-06-26 (entity last revised)  
**Entity name:** `vernier_pos_finder`  
**Purpose:** Decodes the 64-bit thermometer-code output of `tdc_unit_cont` (jta_odelay.vhd) into a **6-bit TDC position** using a 5-stage pipelined priority encoder. Also passes through a 16-bit fine count (coarser timestamp) with matched pipeline delay.

### Pipeline Architecture (5 stages, all clocked at TDC clock)

| Stage | Operation |
|-------|-----------|
| 1 | Slice 64-bit input into 16 groups of 4 bits; register with fine_count |
| 2 | For each 4-bit slice: check if all-1 → set `slice_all_1[i]`; pipeline fine_count |
| 3 | Priority encode `slice_all_1[15:0]` → `pos_high[3:0]` (MSB nibble of position); flag `internal_valid`; pipeline slices |
| 4 | Select the 4-bit data slice at `pos_high` index → `selected_data_slice`; pipeline fine_count + valid |
| 5 | Decode `selected_data_slice` → 2-bit LSB → concatenate with `pos_high` → `TDC_POS[5:0]`; assert `TDC_VALID` |

### Output Fields

| Signal | Width | Description |
|--------|-------|-------------|
| `TDC_POS` | 6 bits | Thermometer transition position: `pos_high[3:0] & pos_low[1:0]` → range 0–63 |
| `TDC_OFFSET` | 16 bits | Fine count passthrough (pipeline-aligned) |
| `TDC_VALID` | 1 bit | '1' if a valid edge was found in the thermometer code |
| `DIAG_SLICER` | 16 bits | `slice_all_1` snapshot (diagnostic) |
| `DIAG2` | 16 bits | Packed debug: `{valid[15], 000[14:12], pos_high[7:4], diag_flag[11:8], selected_slice[3:0]}` |
| `DIAG_RAW` | 64 bits | Frozen copy of last valid raw thermometer code (diagnostic) |

### Priority Encoding Detail

The 16-bit `slice_all_1` vector is a thermometer code of 4-bit groups — `slice_all_1[i]=1` when all 4 bits of group `i` are 1 (signal hasn't propagated past group `i`). The case statement in stage 3 finds the highest group where the thermometer transitions from 1 to 0, e.g.:
- `0111_1111_1111_1111` → pos_high = 15 (transition in top group)
- `0000_0000_0000_0001` → pos_high = 0
- `0000_0000_0000_0000` → only valid if group 0 is non-zero (partial transition in first 4 bits)

The 4-bit `selected_data_slice` is then decoded using a lookup that maps the bit pattern to a 2-bit LSB (0–3), giving 64 total positions.

### Notes

- Valid only asserted when thermometer shows a single clean transition; pathological codes (all-0, multi-transition) set `TDC_VALID='0'`
- Stage 5 has commented-out guards for the "all-ones" and "all-zeros" edge cases — these special cases are now handled only by the stage 2/3 logic
- Total pipeline latency: **5 clock cycles** at TDC clock rate
- Fine count is held in 4-deep pipeline to remain aligned with TDC_POS output

---

_Source: `MTRG_git/MAIN_FPGA/trunk/Source/` — all files read 2026-04-24_  
_See also: `vhdl/MTRG_top.md`, `vhdl/MTRG_trig_algo_support.md`, `vhdl/MTRG_mstr_mach.md`, `deep_fpga_MTRG_MAIN.md`_
