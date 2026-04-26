# MTRG Generated_top.vhd — Top-Level Integration

Stability: C3 - Structural / stable

**Source:** `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/Generated_top.vhd`  
**Lines:** 6,286 ✅ verified 2026-04-24 - Generated_top.vhd:L1 (`wc -l`)  
**Entity:** `trigger_top` ✅ verified 2026-04-24 - Generated_top.vhd:L33  
**Architecture:** `trigtop` ✅ verified 2026-04-24 - Generated_top.vhd:L200

**Note:** Despite the filename `Generated_top.vhd`, this is **not** auto-generated code. The file comment calls it `Spreadsheet_top.vhd` ("the file READ by the code generating spreadsheet") and was hand-authored by John Anderson. It is the true structural top of the DGS Master Trigger FPGA design.

---

## Table of Contents

1. [Purpose](#purpose)
2. [Entity Ports](#entity-ports)
3. [Clock Infrastructure](#clock-infrastructure)
4. [Firmware Type Codes](#firmware-type-codes)
5. [Component Instance Map](#component-instance-map)
6. [Trigger Algorithm Slot Assignments](#trigger-algorithm-slot-assignments)
7. [Trigger Pipeline Overview](#trigger-pipeline-overview)
8. [Veto System](#veto-system)
9. [Monitor FIFO Summary](#monitor-fifo-summary)
10. [Inline Logic (not sub-module)](#inline-logic-not-sub-module)
11. [Code Date & Revision](#code-date--revision)
12. [Key Signals](#key-signals)
13. [See Also](#see-also)

---

## Purpose

This file is the structural **glue layer** of the MTRG firmware. It:
- Declares all top-level I/O pins (SERDES links A–H, L, R, U; VME; AUX ports; NIM; clocks)
- Instantiates every functional sub-module and wires them together
- Implements inline logic for clocking, pad buffering, AUX I/O direction, monitor FIFO control, NIM delay, veto aggregation, and diagnostic ILA hookup
- Decodes VME registers into named signal bit-fields for use throughout the design

---

## Entity Ports

| Group | Signals | Description |
|-------|---------|-------------|
| SERDES Links A–H | `LINKx_RCLK`, `LINKx_RCLK_IO`, `LINKx_RX[17:0]`, `LINKx_TX[17:0]` | 8 digitizer-facing SERDES links (A=1, B=2, … H=8) |
| SERDES Links L/R/U | Same pattern | L=9 (Link L, inter-master), R=10, U=11 (MYRIAD or remote master) |
| Grouped link controls | `LINK_DEN`, `LINK_REN`, `LINK_LINE_LE`, `LINK_LOCAL_LE`, `LINK_TPWRDN`, `LINK_RPWRDN`, `LINK_SYNC`, `LINK_LOCK` | 11-element vectors controlling SERDES transceiver hardware |
| VME Interface | `LOC_ADDR[15:0]`, `LOC_DATA[15:0]` (bidir), `VME_CS0/1/2`, `VME_RNW`, `VME_STB`, `BUF_SYSCLK`, `LACK_OUT` | 16-bit VME A16/D16 interface |
| AUX I/O | `AUX_A_DIR[3:0]`, `AUX_B_DIR[3:0]`, `AUX_PORT_A[7:0]`, `AUX_PORT_B[7:0]`, `NIM_IN1/2`, `NIM_OUT1/2`, `LED[12:2]` | Front-panel auxiliary digital I/O + NIM + LEDs |
| Clock control | `CLK_CTL[29:0]`, `LOGIC_CLOCK`, `VME_CLOCK`, `CLK_SRC_SEL` | 3× clock buffer skew control (chips #1–3) + clock mux |
| LVDS drivers | `DRV_CTL[8:0]` | Pre-emphasis control for 3 LVDS driver chips |
| Misc | `FAST_STROBE`, `SUMCOPY[3:0]`, `SPARE_LVDS[8:1]`, `FPGA2FPGA[9:0]`, `XXLVDS[1:0]`, `MASTER_RESET`, `P2D[23:22]`, `TEST_POINT[11:1]` | CPLD interface, diagnostics, board-to-board |
| Rev-D only | `SUM_CONN_BUF[7:0]`, `U61_1_DIR`, `U61_1_OE` | Expansion connector signals specific to Revision D master trigger boards |

**Generics:** `MAIN_ILA_ENABLE : integer`, `TDC_ILA_ENABLE : integer` — compile-time ILA enable flags.

---

## Clock Infrastructure

| Signal | Description |
|--------|-------------|
| `mclk` | 50 MHz master clock throughout design (from DCM on `LOGIC_CLOCK`) |
| `mclk_2x` | 100 MHz (×2 DCM output) — used for TDC FIFO reads, 250 MHz internally in TDC chain |
| `inv_mclk` | 180° inverted 50 MHz — used for 2-phase carry-chain TDC |
| `xVME_CLOCK` | 50 MHz oscillator-only clock (used for VME and `link_init`, never sourced from SERDES) |
| `xLINKx_RCLK` | Per-link recovered 50 MHz receive clocks from each SERDES |
| `ILA0_CLK` | Muxed clock for ILA0 debug core |
| `DCM_50MHZ_CLK`, `DCM_100MHZ_CLK`, `INV_50MHZ_CLK` | Raw DCM outputs |

The master clock source is selected by `CLK_SRC_SEL` driving the ICS581 mux (`LOGIC_CLOCK` can be either the 50 MHz oscillator or `LINKA_RCLK`).

CLK_CTL bit groups:
- `CLK_CTL[7:0]` → clock buffer chip #1 (links A, B, C, D) skew pairs
- `CLK_CTL[15:8]` → chip #2 (links E, F, G, H)
- `CLK_CTL[23:16]` → chip #3 (links L, R, U, spare)
- `CLK_CTL[25:24/27:26/29:28]` → FS and TEST pins for chips #1/2/3

---

## Firmware Type Codes

Defined as constants for the `CODE_TYPE` field readable via VME:

| Code | Value | Firmware |
|------|-------|---------|
| `cCodeType_DGS_MT` | 4 | **DGS Master Trigger** (this file) |
| `cCodeType_DGS_RTR` | 6 | DGS Router |
| `cCodeType_DGS_DIG` | 12 | DGS Digitizer |
| `cCodeType_MyRIAD` | 11 | MyRIAD trigger expansion module |
| `cCodeType_DFMA_MT` | 5 | DSSD Master Trigger |
| Other | 0–15 | Proto, GRETINA variants, Data Generator, etc. |

✅ verified 2026-04-24 - Generated_top.vhd:L205-219 (all type code constants confirmed)

---

## Component Instance Map

All sub-module instantiations in `trigtop`, in order of appearance:

| Instance | Component | Lines | Description |
|----------|-----------|-------|-------------|
| `U1` | `timestamp` | L3161 | 48-bit timestamp counter; slave sync from Link L |
| `U2` | `mstr_mach` | L3174 | Master state machine — full 20-frame TTCL output engine |
| `U3` | `LINK_TX_BLOCK` | L3244 | DC-balanced SERDES fan-out to all 11 links |
| `U4` | `link_init` | L3261 | SERDES lock/init sequencer (runs on `xVME_CLOCK`) |
| `U5` | `LINK_LRU_RX` | L3277 | Latches RX data from links L, R, U into `mclk` domain |
| `TRIG_LOGIC1` | `cpld_trig` | L3359 | Algo 1: CPLD fast-sum trigger (thin wrapper over `trig_algo_support`) |
| `TRIG_LOGIC2` | `sum_hits_X` | L3413 | Algo 2: Sum-of-X multiplicity trigger |
| `TRIG_LOGIC3` | `sum_hits_X` | L3471 | Algo 3: Sum-of-Y multiplicity trigger (same component, Y-plane links) |
| `TRIG_LOGIC4` | `sum_hits_XY` | L3527 | Algo 4: Sum-of-X AND Sum-of-Y coincidence trigger |
| `TRIG_LOGIC5A` | `cpld_trig` | L3659 | Algo 5A: CPLD fast-sum trigger (second instance for Algo 5) |
| `TRIG_LOGIC5B` | `local_trig_coinc` | L3706 | Algo 5B: Local-vs-local coincidence trigger (type `0x54`) |
| `TRIG_LOGIC6` | `REMOTE_MASTER_TRIG_SUPPORT` | L3768 | Algo 6: Remote trigger from Link L (inter-master) |
| `TRIG_LOGIC7` | `REMOTE_MASTER_TRIG_SUPPORT` | L3842 | Algo 7: Remote trigger from Link R |
| *(no TRIG_LOGIC8 instance)* | — | L3912 | **Not a component instance** — L3912 is inline mux logic routing TRIG_LOGIC8A/8B outputs to `TRIGGER_FIFO_OUT(8)` via `TRIG_LOGIC_8_ALGO_SEL` |
| `TRIG_LOGIC8A` | `MYRIAD_TRIGGER` | L3994 | Algo 8A: MyRIAD-based trigger (Link U, MYRIAD mode) |
| `TRIG_LOGIC8B` | `REMOTE_MASTER_TRIG_SUPPORT` | L4065 | Algo 8B: Remote master from Link U (generic mode) |
| `TRIGGER_COLLECTION` | `trig_collect` | L4142 | Trigger collection mux — arbitrates 8 algo FIFOs into single stream for mstr_mach |
| `U20` | `registers` | L4202 | VME register block (~120 R/O + R/W regs, 3 lookup RAMs, 8+8 monitor FIFOs) |
| `LINK_L_RECEIVER` | `SERDES_RX_Mach` | L4619 | Link L SERDES receiver — full 20-frame FSM |
| `LINK_R_RECEIVER` | `SERDES_RX_Mach` | L4692 | Link R SERDES receiver |
| `LINK_U_MYR_RECEIVER` | `MYRIAD_RCV_MACH` | L4764 | Link U MyRIAD receiver (active when `LINK_U_IS_TRIGGER_TYPE=0`) |
| `LINK_U_TRIG_RECEIVER` | `SERDES_RX_Mach` | L4782 | Link U generic trigger receiver (active when `LINK_U_IS_TRIGGER_TYPE=1`) |
| `TDC1` | `tdc_chain_cont` | L5559 | 4-phase carry-chain TDC controller; 8-word event packet; autosample FSM |
| `TRIG_MON_COLLECTOR` | `trig_mon_collect` | L5591 | Trigger monitor FIFO collector → Monitor FIFO 7 |

✅ verified 2026-04-24 - Generated_top.vhd: all instance names and line numbers confirmed by grep

**Note on Link U mux:** `MYRIAD_RCV_MACH_RESET <= LINK_U_IS_TRIGGER_TYPE`. When this bit is set (from `reg_TRIG_ALGO_MUX_SEL(2)`, addr 0x021C — **correction**: earlier stated `reg_LINK_LRU_CTL`, which is wrong), the MYRIAD receiver is held in reset and Link U is used as a generic master-master link via `LINK_U_TRIG_RECEIVER`. ✅ verified 2026-04-24 - Generated_top.vhd:L1605,L4760

---

## Trigger Algorithm Slot Assignments

| Slot | Instance | Component | Trigger Type | Input Source |
|------|----------|-----------|-------------|--------------|
| 1 | `TRIG_LOGIC1` | `cpld_trig` | CPLD fast sum | `FAST_STROBE` pin via CPLD |
| 2 | `TRIG_LOGIC2` | `sum_hits_X` | Sum-of-X | Links A–H (X-plane) |
| 3 | `TRIG_LOGIC3` | `sum_hits_X` | Sum-of-Y | Links A–H (Y-plane) |
| 4 | `TRIG_LOGIC4` | `sum_hits_XY` | Sum-of-XY | Both planes simultaneously |
| 5A | `TRIG_LOGIC5A` | `cpld_trig` | CPLD fast sum (alt) | `FAST_STROBE` |
| 5B | `TRIG_LOGIC5B` | `local_trig_coinc` | Local coincidence | Other algo ACKs, mask configurable |
| 6 | `TRIG_LOGIC6` | `REMOTE_MASTER_TRIG_SUPPORT` | Remote master | Link L |
| 7 | `TRIG_LOGIC7` | `REMOTE_MASTER_TRIG_SUPPORT` | Remote master | Link R |
| 8A | `TRIG_LOGIC8A` | `MYRIAD_TRIGGER` | MyRIAD | Link U (MYRIAD mode) |
| 8B | `TRIG_LOGIC8B` | `REMOTE_MASTER_TRIG_SUPPORT` | Remote master | Link U (generic mode) |

Algorithms 5A+5B and 8A+8B are mutually exclusive runtime selections, controlled by VME registers (`ALGO_5_SELECT`, `LINK_U_IS_TRIGGER_TYPE`). The `trig_collect` component arbitrates all 8 slots (Fifo1–8) into the command stream.

---

## Trigger Pipeline Overview

```
SERDES Links A–H
     ↓
eight_mt_channel  (8 inputs → decoded data + X/Y sums)
     ↓
TRIG_LOGIC1–8     (individual trigger algorithms)
     ↓
TRIGGER_COLLECTION (trig_collect — 8-algo FIFO arbiter)
     ↓
mstr_mach          (20-frame TTCL master state machine)
     ↓
LINK_TX_BLOCK      (DC-balanced fan-out → all 11 SERDES TX)
```

Remote triggers arrive via `LINK_L/R/U_RECEIVER` → `TRIG_LOGIC6/7/8` and feed back into the same `trig_collect` stream.

---

## Veto System

The veto system was redesigned on 2016-04-01 (JTA: "no, not April Fool's") to allow per-algorithm veto specificity.

**`TRIGGER_VETOES[8:1]`** — one bit per algorithm; each algorithm independently responds to different veto sources.

**`SYSTEM_VETO_STATE[15:0]`** — global veto state word (readable via VME), format:

```
 15    14    13    12    11    10    09    08    07    06    05    04    03    02    01    00
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
|RM U |RM R |RM L |SW V |V RAM|MON7 |GLOB | NIM |Algo7|Algo6|Algo5|Algo4|Algo3|Algo2|Algo1|Algo0|
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
```

Veto sources:
- **NIM veto** (`ENBL_NIM_VETO`) — NIM input
- **VETO RAM** (`VETO_FROM_VETO_RAM`) — target wheel address lookup
- **MON7 FIFO veto** (`MON7_VETO_REQUEST`) — trigger system backpressure
- **Global throttle** (`GLOBAL_THROTTLE_REQUEST`) — algo FIFO more than half full
- **Remote master veto** from Links L/R/U (`VETO_FROM_REMOTE_MASTER_L/R/U`) — added 2021-06-16
- **Software veto** (`SOFTWARE_VETO`) — VME-writable

`ANY_TRIGGER_VETO_TO_REMOTE` excludes the case where veto originates solely from a remote master being looped back.

Per-algorithm veto selectivity is configured via `reg_TRIG_VETO_SELECT_A–H` (default `0x000F` = all vetoes apply).

---

## Monitor FIFO Summary

| FIFO | Source | Data / Trigger condition |
|------|--------|--------------------------|
| MON1 | Embedded in `mstr_mach` | Master state machine diagnostic data |
| MON2 | Embedded in `LINK_TX_BLOCK` | TX link output monitoring |
| MON3 | Inline logic | Configurable: algo FIFO flags, FAST_STROBE, veto, Frame 12/14/16 sent flags; OR throttle-change state machine (4-word: THROTTLE_MAP + 3-word TS) |
| MON4 | Inline logic | Multiplicity data (GLOBAL_X_TOTAL, GLOBAL_Y_TOTAL, CPLD sum); WE triggers configurable per bit 15:6 of `reg_MON4_FIFO_SEL` |
| MON5 | Inline logic | GLOBAL_Y_TOTAL or TS[15:0]; WE on Y/X nonzero or force |
| MON6 | Inline logic | SERDES link L, R, or U raw RX data |
| MON7 | `trig_mon_collect` | TDC + trigger history data from all 8 algorithm shadow FIFOs |
| MON8 | Inline logic | Trigger FIFO RE debug signals |

MON3 has a secondary throttle-change capture state machine (`THROTTLE_MON_STATES`: IDLE→CAPTURE_THROTTLE_MAP→TS1→TS2→TS3→WE_OFF) that fires on any edge of `GLOBAL_THROTTLE_REQUEST`.

---

## Inline Logic (not sub-module)

The following functional blocks are implemented directly in the `trigtop` architecture rather than as component instances:

- **DCM instantiation** — DCM_BASE generates 50 MHz, 100 MHz, inverted 50 MHz from `LOGIC_CLOCK`
- **IBUF / OBUF / IOBUF pad buffers** — all SERDES clocks, AUX_PORT_A/B (IOBUFs), AUX_A/B_DIR (OBUFs)
- **AUX I/O direction control** — `xAUX_A_DIR[3:0]` bits drive IOBUF direction; `A_3_0_DIR` / `A_7_4_DIR` / `B_3_0_DIR` / `B_7_4_DIR` from register bits
- **NIM input pipeline** — `xxNIM_IN1` / `xxxNIM_IN1` / `xxxxNIM_IN1` edge-detection chain; similarly for NIM_IN2 and FAST_STROBE
- **NIM programmable delay** (`EN_NIM1_DELAY`, `EN_NIM2_DELAY`) — selectable delay path for NIM inputs before triggering logic
- **VETO/TRIG/SWEEP RAM address mux** — `ENCODER_ADDRESS_FROM_AUX`, `INTERNAL_ENCODER_COUNTER`, `ENCODER_TEST_SUM` muxed by `ENCODER_SOURCE_SELECT` register
- **Target wheel address sampling** — `SAMPLED_LOCAL_TRIG_MON_DET_DATA` / `SAMPLED_LOCAL_TRIG_MON_XTRA_DATA` captured at trigger time
- **Frame 12/14/16/17 state machines** — `FRAME_12_STATES`, `FRAME_14_STATES`, `FRAME_16_STATES`, `FRAME_17_STATES` (IDLE/WAIT_TIME_MATCH/WAIT_FLAG_ACK) manage inter-trigger command injection
- **Rate counter state machine** (`TRIG_RATE_STATES`: RESET/WAIT_TS_MATCH/ACCUMULATE/HOLD) — synchronized capture of trigger rate counters
- **Monitor FIFO 3/4/5/8 control processes** — inline `process` blocks for WE generation and data mux
- **LED and test point assignments** — `LED4`–`LED12`, `xTEST_POINT[11:1]`
- **SUMCOPY** — output to CPLD, redefined 2008-10-29 to carry VME address bits (helps CPLD decode running multiplicity)
- **ILA diagnostic core hookup** — `ILA_trig0` 64-bit signal composed at top level from AUX_INPUT_STATE, trigger flags, state monitors

---

## Code Date & Revision

From signal declarations (latest compiled firmware values visible in VHDL):

| Register | Addr | Default | Meaning |
|----------|------|---------|---------|
| `reg_CODE_DATE` | 0x0158 | `0x1127` | Date code (November 27) |
| `reg_CODE_REVISION` | 0x015C | `0x04A9` | Revision 04A9 |

✅ verified 2026-04-24 - Generated_top.vhd:L610-611

---

## Key Signals

| Signal | Type | Description |
|--------|------|-------------|
| `mclk` | std_logic | 50 MHz master clock throughout design |
| `mclk_2x` | std_logic | 100 MHz clock |
| `TIME_STAMP_BUS` | [47:0] | Live 48-bit timestamp from `timestamp` module |
| `CONTROL_DATA` | [15:0] | 20-frame TTCL command word from `mstr_mach` |
| `xLINK_TX` | JTA_11×18 | DC-balanced TX to all 11 SERDES links |
| `xLINK_RX` | JTA_11×18 | Raw RX from all 11 SERDES links |
| `TRIGGER_FIFO_OUT` | JTA_8×16 | Output from each algo trigger FIFO to `trig_collect` |
| `TRIGGER_FIFO_FLAG` | [8:1] | Data-available flags from 8 algo FIFOs |
| `TRIGGER_VETOES` | [8:1] | Per-algorithm veto inputs |
| `SYSTEM_VETO_STATE` | [15:0] | Aggregated veto status word |
| `RAW_NONVETOED_TRIG_ACK` | [8:1] | Trigger ack irrespective of enable, honors veto |
| `ENABLED_TRIG_ACK` | [8:1] | Honors enable, ignores veto (for rate counting) |
| `ENABLED_NONVETOED_TRIG_ACK` | [8:1] | Honors both enable and veto (actual trigger issuance) |
| `GLOBAL_X_TOTAL` / `GLOBAL_Y_TOTAL` | [15:0] | Summed X/Y plane multiplicities from `eight_mt_channel` |
| `ENCODER_ADDRESS` | [9:0] | Target wheel position (10 bits) |
| `FRAME_12/14/16/17_REQ_FLAG` | std_logic | Request flags for inter-trigger command frames |

---

## See Also

- [MTRG_top.md](MTRG_top.md) — earlier analysis of `top.vhd` (simpler wrapper)
- [MTRG_mstr_mach.md](MTRG_mstr_mach.md) — U2: master state machine
- [MTRG_support_modules.md](MTRG_support_modules.md) — U1 (timestamp), U3 (link_tx_block), etc.
- [MTRG_registers.md](MTRG_registers.md) — U20: full VME register map
- [MTRG_SERDES_RX_Mach.md](MTRG_SERDES_RX_Mach.md) — LINK_L/R/U_RECEIVER
- [MTRG_MYRIAD_RCV_MACH.md](MTRG_MYRIAD_RCV_MACH.md) — LINK_U_MYR_RECEIVER
- [MTRG_MYRIAD_TRIGGER.md](MTRG_MYRIAD_TRIGGER.md) — TRIG_LOGIC8A
- [MTRG_tdc_chain_cont.md](MTRG_tdc_chain_cont.md) — TDC1
- [MTRG_eight_mt_channel.md](MTRG_eight_mt_channel.md) — U10
- [MTRG_local_trig_coinc.md](MTRG_local_trig_coinc.md) — TRIG_LOGIC5B
- [MTRG_trig_algo_support.md](MTRG_trig_algo_support.md) — shared trigger algorithm base
- [MTRG_AUX_IO.md](MTRG_AUX_IO.md) — AUX port mux and encoder logic (implemented inline here, not as sub-module)
- [deep_fpga_MTRG_MAIN.md](../deep_fpga_MTRG_MAIN.md) — overall MTRG FPGA architecture overview
- [fpga.md](../fpga.md) — VHDL analysis index

---

*Analyzed 2026-04-24. Fact-verified 2026-04-24: entity/arch names, line count, firmware type codes, all component instance names and exact line numbers, reg_CODE_DATE/REVISION defaults. Correction: LINK_U_IS_TRIGGER_TYPE source was `reg_LINK_LRU_CTL` (wrong) → corrected to `reg_TRIG_ALGO_MUX_SEL(2)` (L1605). Correction: TRIG_LOGIC8 entry was listed as a component instance (wrong) → corrected to inline mux logic at L3912. Source: Generated_top.vhd.*
