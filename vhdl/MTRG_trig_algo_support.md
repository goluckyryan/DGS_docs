# MTRG: trig_algo_support.vhd — Generic Trigger Algorithm Support

**Source:** `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/trig_algo_support.vhd` (481 lines)  
Stability: C3 - Structural / stable  
Last verified: 2026-04-23

---

## Table of Contents

- [Overview](#overview)
- [Key Ports](#key-ports)
  - [Inputs](#inputs)
  - [Outputs](#outputs)
- [Trigger Decision Logic (ACK State)](#trigger-decision-logic-ack-state)
- [Prescaler](#prescaler)
- [Holdoff (added 2025-10-22)](#holdoff-added-2025-10-22)
- [FIFO Record Format](#fifo-record-format)
- [Throttle Logic](#throttle-logic)
- [FIFOs Used](#fifos-used)
- [State Machine States](#state-machine-states)
- [Cross-References](#cross-references)

---

## Overview

`trig_algo_support` is the **shared base component instantiated by every MTRG trigger algorithm** (e.g., `cpld_trig`, `sum_hits_X`, `sum_hits_XY`, `local_trig_coinc`, `MYRIAD_TRIGGER`). It encapsulates all the boilerplate infrastructure so that each trigger algorithm only needs to implement its own condition logic, then delegate everything else to this module.

**What it provides:**
1. **Trigger algorithm FIFO** — 1024-deep FIFO that queues accepted trigger records
2. **Shadow TRIG_MON FIFO** — separate FIFO (512 write × 1024 read depth) for trigger monitoring data with extra payload (target wheel state, etc.)
3. **Event counter** — 8-bit saturating counter tracking how many events are queued
4. **Prescaler** — skip every N-th trigger (configurable, 8-bit counter)
5. **Holdoff** — after firing, veto self for N×20 ns ticks (added 2025-10-22)
6. **Throttle request** — asserts `ALGO_THROTTLE_REQUEST` if either FIFO is more than half-full (added 2014-12-11 by MBO)

---

## Key Ports

### Inputs

| Port | Width | Description |
|------|-------|-------------|
| `CLK` | 1 | Board-wide 50 MHz clock |
| `RST` | 1 | Global reset |
| `TRIGGER_OCCURRED` | 1 | Algorithm condition satisfied (from caller) |
| `TRIGGER_TYPE_CODE` | variable | 8-bit type code for this algorithm (e.g. `0x55`, `0x5A`) |
| `TIME_STAMP_BUS` | 48 | Running 48-bit timestamp |
| `TRIG_DIST_MASK` | 8 | Which Router links receive this trigger (from VME register) |
| `TRIGGER_ENABLE` | 1 | Algorithm enable bit (from TRIG_MASK VME register) |
| `TRIGGER_VETO` | 1 | Global veto (if set: no trigger is loaded to FIFO) |
| `TRIGGER_HOLDOFF` | 16 | Holdoff count in 20 ns ticks (added 2025-10-22) |
| `TRIG_HOLDOFF_ENBL` | 1 | 0=no holdoff, 1=enable holdoff (added 2025-10-22) |
| `TRIGGER_PRESCALE` | 16 | Prescale config: bit 15 = enable, bits 7:0 = count |
| `TRIG_FIFO_RE` | 1 | Read enable from master trigger collector machine |
| `TRIG_FIFO_ACK` | 1 | Acknowledge from collector (event fully read) |
| `LOCAL_TRIG_MON_DET_DATA` | 16 | Detector state at trigger time (e.g. target wheel) — monitor FIFO only |
| `LOCAL_TRIG_MON_XTRA_DATA` | 16 | Extra state data — monitor FIFO only |

### Outputs

| Port | Width | Description |
|------|-------|-------------|
| `TRIG_FIFO_OUT` | 16 | Data word from the algorithm FIFO |
| `EVENT_AVAILABLE` | 1 | Non-zero events queued in FIFO |
| `RAW_NONVETOED_TRIG_ACK` | 1 | Trigger occurred, not vetoed, regardless of enable (used for matrix trigger cross-signaling) |
| `ENABLED_TRIG_ACK` | 1 | Trigger occurred while enabled (veto ignored) — used for rate counting |
| `ENABLED_NONVETOED_TRIG_ACK` | 1 | Trigger occurred while enabled AND not vetoed — actual accepted trigger |
| `ALGO_THROTTLE_REQUEST` | 1 | Either FIFO is >half-full; throttle back input rate |
| `TRIG_MON_FIFO_DATA_OUT` | 16 | Data from the shadow monitor FIFO |
| `MON_EVENT_AVAILABLE` | 1 | Shadow monitor FIFO has data |

---

## Trigger Decision Logic (ACK State)

The ACK state implements this truth table ✅ verified 2026-04-23 — `trig_algo_support.vhd:L263-339` (ACK state case):

| TRIGGER_ENABLE | TRIGGER_VETO | FIFO Loaded? | RAW_NONVETOED | ENABLED | ENABLED_NONVETOED |
|:-:|:-:|:-:|:-:|:-:|:-:|
| 0 | 0 | No | ✓ | — | — |
| 0 | 1 | No | — | — | — |
| 1 | 0 | **Yes** | ✓ | ✓ | ✓ |
| 1 | 1 | No | — | ✓ | — |

`RAW_NONVETOED_TRIG_ACK` fires whenever: enabled=0, veto=0 (to notify matrix triggers even when algorithm is disabled).  
`ENABLED_TRIG_ACK` fires whenever: enabled=1, regardless of veto (for rate counting).  
`ENABLED_NONVETOED_TRIG_ACK` fires (and FIFO is written) only when: enabled=1 AND veto=0.

---

## Prescaler

Controlled by `TRIGGER_PRESCALE[15:0]`:
- Bit 15 = enable prescaling
- Bits 7:0 = skip count N

When enabled, one trigger is accepted per (N+1) occurrences. The prescale counter decrements each time a trigger is vetoed by prescaling, resets to N after each accepted trigger. ✅ verified 2026-04-23 — `trig_algo_support.vhd:L218-244` (WAIT_TRIG prescale update), L296-312 (ACK prescale handling)

---

## Holdoff (added 2025-10-22)

When `TRIG_HOLDOFF_ENBL='1'`, after each accepted trigger the module enters holdoff for `TRIGGER_HOLDOFF` × 20 ns. During holdoff, `TRIGGER_OCCURRED` is ignored (self-veto). Holdoff count decrements every clock cycle (50 MHz → 20 ns per tick). ✅ verified 2026-04-23 — `trig_algo_support.vhd:L104,L215-261` (holdoff state logic)

---

## FIFO Record Format

Each accepted trigger writes **4 words** to the algorithm FIFO ✅ verified 2026-04-23 — `trig_algo_support.vhd:L356-380` (FILL1–FILL4 states):

| Word | Algorithm FIFO | Monitor FIFO (32-bit write pairs) |
|------|---------------|-----------------------------------|
| 1 | `TRIGGER_TYPE_CODE[7:0] & TRIG_DIST_MASK[7:0]` | Same + `LATCHED_TIMESTAMP[47:32]` (32-bit write) |
| 2 | `LATCHED_TIMESTAMP[47:32]` | `LATCHED_TIMESTAMP[31:0]` (32-bit write) |
| 3 | `LATCHED_TIMESTAMP[31:16]` | `LOCAL_TRIG_MON_DET_DATA[15:0] & LOCAL_TRIG_MON_XTRA_DATA[15:0]` |
| 4 | `LATCHED_TIMESTAMP[15:0]` | (no further write) |

The monitor FIFO receives the same 4 entries but in 32-bit write pairs (512 depth × 32 bits in → 1024 depth × 16 bits out). The monitor FIFO also gets detector state (`DET_DATA`, `XTRA_DATA`) that the algorithm FIFO does not.

---

## Throttle Logic

`ALGO_THROTTLE_REQUEST` is asserted when either the algo FIFO `ALGO_PROG_FULL` or the monitor FIFO `MON_ALMOST_FULL` is set, but only when `TRIGGER_ENABLE='1'` (a disabled algorithm FIFO could spuriously assert PROG_FULL while in reset). ✅ verified 2026-04-23 — `trig_algo_support.vhd:L232-238`

`PROG_FULL_THRESH` is set to `b"1000000000"` = 512 (half the 1024-depth FIFO). ✅ verified 2026-04-23 — `trig_algo_support.vhd:L95`

---

## FIFOs Used

| FIFO Instance | Component | Depth (in/out) | Width (in/out) | Notes |
|---------------|-----------|---------------|----------------|-------|
| `U1` (algo FIFO) | `FIFO_IND_16Wx1024D_STD` | 1024 | 16-bit in, 16-bit out | Single clock; prog_full threshold=512 |
| `TRIG_MON_FIFO` | `FIFO_FWFT_32WX512DIN_16WX1024DOUT_SEPCLK` | 512 write / 1024 read | 32-bit in, 16-bit out | Separate clocks (write: CLK, read: TRIG_MON_FIFO_RD_CLK); replaced older FIFO 2021-03-02 |

---

## State Machine States

| State | Action |
|-------|--------|
| `INIT` | Clear all outputs, initialize holdoff/prescale, go to WAIT_TRIG |
| `WAIT_TRIG` | Wait for `TRIGGER_OCCURRED`; manage prescale enable; update throttle |
| `ACK` | Evaluate enable/veto/prescale/holdoff; assert appropriate ACK signals; latch timestamp |
| `FILL1`–`FILL4` | Write 4 words to both FIFOs (algo FIFO 4 words, monitor FIFO 3×32-bit writes) |
| `SIG` | Assert `TRIG_BIT` for one cycle to notify event counter, return to WAIT_TRIG |
| `MON_FILL1`–`MON_FILL3` | Write only to monitor FIFO (algorithm FIFO not written) — used when algorithm not enabled but monitoring enabled |

---

## Cross-References

- `MTRG_sum_hits_X.md` — uses this module as U2 for the X-multiplicity trigger
- `MTRG_local_trig_coinc.md` — uses this module as U2 for the local coincidence trigger
- `MTRG_MYRIAD_TRIGGER.md` — uses this module as shared base
- `deep_fpga_MTRG_MAIN.md` — overall MTRG trigger architecture
- `VME_registers.md` — TRIG_MASK, TRIGGER_PRESCALE, AUX_IO registers
