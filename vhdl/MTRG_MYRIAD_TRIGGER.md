# MTRG: `MYRIAD_TRIGGER.vhd` — MγRIAD Trigger Algorithm
Stability: C3 - Structural / stable

**Module:** `MYRIAD_TRIGGER`  
**File:** `FPGA/Firmware_Tags/MasterTrigger/20220705/Source/MYRIAD_TRIGGER.vhd`  
**Author:** John T. Anderson (ANL), created 2011-09-03; revised 2015-06-01 (algorithm overhaul), 2015-10-01 (monitoring FIFO)  
**Part of:** MTRG Main FPGA (Virtex-4 / Kintex UltraScale in Vivado port)  
**Clock:** 50 MHz board-wide clock  
**Source tag used:** `20220705` (most recent tag available)

---

## Purpose

Receives decoded MγRIAD trigger signals from `MYRIAD_RCV_MACH` and converts them into MTRG trigger FIFO entries, with:
- Programmable delay (delay line for cross-detector timing alignment)
- Optional coincidence gating with other MTRG trigger algorithms
- Selectable raw vs. gated MγRIAD trigger source
- Selectable timestamp mode
- Prescale, veto, enable, and distribution mask (standard MTRG trigger infrastructure)

This is a **standard MTRG trigger algorithm** in the same family as SumX, SumY, CPLD fast-sum, etc. It uses the shared `trig_algo_support` component for all FIFO and acknowledge logic.

---

## Algorithm Overview (Post-May-2015 Design)

The algorithm operates in two parallel paths that can be combined via overlap logic:

### Path 1 — MγRIAD trigger path
1. Wait for `MYRIAD_DATA_LOCK = '1'` (MγRIAD receiver is synchronized)
2. Watch for `RAW_TRIGGER` or `GATED_TRIGGER` pulse (selected by `MYRIAD_TRIG_TYPE_SELECT`)
3. On trigger arrival: latch `TIME_STAMP_BUS` → push into **Pre-FIFO** (48-bit timestamp FIFO); pulse `TRIG_FLAG_IN` into a **delay line** (`DELAY_LINE` component, programmable via `REG_TRIG_DELAY`)
4. After `REG_TRIG_DELAY` clocks: `MYRIAD_DELAYED_TRIGGER` falls out of the delay line; simultaneously `DELAYED_TIME_STAMP` is read from the Pre-FIFO (with a 2-clock pipeline delay to ensure data validity)

### Path 2 — Other trigger path
- `OTHER_TRIGGER_MASK` (7 bits, one per other MTRG algorithm) selects which other algorithm acknowledge signals (`OTHER_NONVETOED_TRIG_ACK[7:1]`) can activate the coincidence logic
- Matching any enabled bit asserts `MASKED_OTHER_TRIGGER`

### Overlap Logic (`overlap_mach` instance)
Shared `overlap_mach` component (same as used in other MTRG algorithms):
- `TRIG_FLAG_1` = `MYRIAD_DELAYED_TRIGGER`
- `TRIG_FLAG_2` = `MASKED_OTHER_TRIGGER`
- Programmable overlap window via `OVERLAP_DELAY` (7-bit)
- Outputs: `MYRIAD_NO_OVERLAP` (MγRIAD only), `OVERLAP_TRIGGER` (both), `OTHER_NO_MYRIAD` (other only)

### Final trigger selection (`MYRIAD_TRIG_SEL_PROC`)
| `OTHER_TRIGGER_MASK` | Final trigger | Subtype |
|----------------------|---------------|---------|
| All zeros (no other algo selected) | `MYRIAD_DELAYED_TRIGGER` directly | `0x78` |
| Any bit set | `OVERLAP_TRIGGER` (coincidence required) | `0x79` |

> When used standalone (mask=0): `RAW_NONVETOED_TRIG_ACK` = `MYRIAD_DELAYED_TRIGGER` (irrespective of veto, used by other algorithms' matrix logic).

---

## Timestamp Modes

Controlled by `MYRIAD_TRIGGER_MODE` register bit:

| Mode | `MYRIAD_TRIGGER_MODE` | Timestamp in trigger record |
|------|-----------------------|------------------------------|
| Default | `'0'` | `TIME_STAMP_BUS` — timestamp when delayed trigger fires (exits delay line) |
| Arrival mode | `'1'` | `DELAYED_TIME_STAMP` from Pre-FIFO — timestamp when MγRIAD message arrived |

The arrival-mode timestamp is latched into the Pre-FIFO at the moment the trigger comes in, then read back out when the delayed version fires. This allows reporting the actual physics event time regardless of the programmed delay.

---

## Trigger Type Codes

| Subtype | Value | Condition |
|---------|-------|-----------|
| MγRIAD standalone | `0x78` | `OTHER_TRIGGER_MASK = 0x00` |
| MγRIAD+overlap | `0x79` | `OTHER_TRIGGER_MASK ≠ 0x00` |

---

## Pre-FIFO Architecture

- **Component:** `FIFO_FWFT_48X64` (48-bit wide, 64 deep, first-word-fall-through)
- **Write:** timestamp pushed at trigger arrival (`PRE_FIFO_WE`)
- **Read:** read enable asserted 2 clocks after `MYRIAD_DELAYED_TRIGGER` (via 3-stage pipeline `PRE_FIFO_RE_PIPE`)
- Purpose: holds the "at-arrival" timestamp until the delayed trigger fires, so that `DELAYED_TIME_STAMP` can be used when `MYRIAD_TRIGGER_MODE='1'`

---

## Internal State Machine (`MYRIAD_TRIG_WE_MACH`)

States: `IDLE` → `WAIT_TRIG` → `LOAD_PRE_FIFO`

| State | Condition | Action |
|-------|-----------|--------|
| `IDLE` | `MYRIAD_DATA_LOCK='1'` | → `WAIT_TRIG`; else hold, keep Pre-FIFO in reset |
| `WAIT_TRIG` | `RAW_TRIGGER='1'` (or `GATED_TRIGGER='1'` if `MYRIAD_TRIG_TYPE_SELECT='1'`) | Latch timestamp, pulse `TRIG_FLAG_IN`, pulse `PRE_FIFO_WE`, → `LOAD_PRE_FIFO` |
| `LOAD_PRE_FIFO` | `RAW_TRIGGER='0'` (trigger de-asserted) | → `WAIT_TRIG` |

The machine holds `LOAD_PRE_FIFO` while the trigger input remains high to prevent double-triggering.

---

## Key Control Ports

| Port | Direction | Description |
|------|-----------|-------------|
| `MYRIAD_DATA_LOCK` | in | From `MYRIAD_RCV_MACH.MACHINE_LOCKED`; must be '1' for any trigger processing |
| `RAW_TRIGGER` | in | MγRIAD raw trigger bit (from `MYRIAD_RCV_MACH`) |
| `GATED_TRIGGER` | in | MγRIAD gated trigger bit (from `MYRIAD_RCV_MACH`) |
| `MYRIAD_TRIG_TYPE_SELECT` | in | '0' = use RAW_TRIGGER; '1' = use GATED_TRIGGER |
| `MYRIAD_TRIGGER_MODE` | in | '0' = delayed-exit timestamp; '1' = arrival timestamp |
| `REG_TRIG_DELAY` | in | 16-bit delay value for timing alignment delay line |
| `OVERLAP_DELAY` | in | 7-bit coincidence overlap window width |
| `OTHER_TRIGGER_MASK` | in | 7-bit mask enabling other MTRG algorithms for coincidence |
| `OTHER_NONVETOED_TRIG_ACK` | in | 7-bit vector; acknowledge signals from other algorithms |
| `TRIGGER_ENABLE` | in | Master enable bit (from VME) |
| `TRIGGER_VETO` | in | When '1', suppresses trigger FIFO writes |
| `TRIGGER_PRESCALE` | in | 16-bit prescale counter (skip N triggers between accepted triggers) |
| `TRIG_DIST_MASK` | in | 8-bit mask for which digitizer/router links receive the trigger |
| `TIME_STAMP_BUS` | in | Running 48-bit MTRG timestamp |
| `MYRIAD_TRIG_TIMESTAMP` | out | 48-bit timestamp latched at trigger arrival (for monitoring) |

---

## Trigger Monitor FIFO

Added October 2015. A shadow FIFO allows a separate state machine to record trigger monitor data (detector state, extra data) at the moment of each trigger. Interface:
- `TRIG_MON_FIFO_RD_CLK`, `TRIG_MON_FIFO_RE` — read clock/enable from monitor state machine
- `TRIG_MON_DET_DATA` — 16-bit detector state (e.g. target wheel position)
- `TRIG_MON_XTRA_DATA` — 16-bit extra state: `GLOBAL_X_TOTAL[7:0] & GLOBAL_Y_TOTAL[7:0]` — the live X+Y multiplicity sums at the moment of trigger ✅ verified 2026-04-22 — `top.vhd:L1185` (`TRIG_MON_XTRA_DATA <= GLOBAL_X_TOTAL(7 downto 0) & GLOBAL_Y_TOTAL(7 downto 0)` — comment: "added 20160307 to put router totals into trigger data stream")
- `TRIG_MON_FIFO_DATA_OUT`, `TRIG_MON_FIFO_FULL`, `TRIG_MON_FIFO_EMPTY` — standard FIFO status

This is passed through to `trig_algo_support` (shared base) which handles the actual shadow FIFO.

---

## Integration in MTRG top.vhd

- MγRIAD is on **Link U** (frame 10 in the 20-frame TTCL command structure)
- MTRG rule: **triggers generated by the MγRIAD algorithm (frame 10) are never sent back to Link U** — MγRIAD receives all other trigger accepts (frames 3–9) but is unaware of its own
- Parallel `GITMO_TRIGGER` module handles Link L (GRETINA/remote master); a VME control bit selects which receiver (`MYRIAD_RCV_MACH` or `GITMO_RCV_MACH`) is held in reset

---

## Other Algorithm Coincidence Matrix

`OTHER_TRIGGER_MASK` bits (7 bits, bits 7:1):

| Bit | Algorithm (acknowledge used) |
|-----|------------------------------|
| 1 | Manual / NIM In (AUX) / TargetWheel trigger (ack #1) |
| 2 | SumX trigger (ack #2) |
| 3 | SumY trigger (ack #3) |
| 4 | SumXY trigger (ack #4) |
| 5 | CPLD fast-sum trigger (ack #5) |
| 6 | GITMO or Remote Master trigger (ack #6) |
| 7 | Remote Master trigger (ack #7) |

All uses `RAW_NONVETOED_TRIG_ACK` from the other algorithm (fires irrespective of enable, respects veto).

---

## Cross-References

- `myriad.md` — MγRIAD hardware, VME register map, front-panel I/O
- `vhdl/MTRG_MYRIAD_RCV_MACH.md` — companion receiver state machine (this module's data source)
- `deep_fpga_MTRG_MAIN.md` — MTRG Main FPGA overview; trigger algorithm instantiation
- `vhdl/RTRG_overlap_mach.md` — `overlap_mach` component shared across trigger algorithms
- `vhdl/MTRG_top.md` — MTRG top-level wiring showing Link U assignment

---

*Created: 2026-04-21 from source `MYRIAD_TRIGGER.vhd` (tag 20220705)*
