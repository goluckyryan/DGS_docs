# sum_hits_X.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/sum_hits_X.vhd_
_Summarized: 2026-04-15_

## Purpose
One Master Trigger algorithm module: fires when the total X-plane multiplicity sum (across all Routers) exceeds a programmable threshold. Implements leading-edge detection (fires once on threshold crossing, waits for sum to fall before re-arming). Delegates all FIFO management, prescaling, veto, enable, and timestamp capture to the generic `trig_algo_support` sub-component.

## Ports
Key algorithm-specific ports:

| Signal | Dir | Width | Description |
|---|---|---|---|
| `SUM_OF_X` | in | 16 | Total X-plane hit count from all Routers |
| `SUM_OF_X_THRESH` | in | 16 | VME-programmable threshold |
| `TRIG_TYPE` | in | 8 | Trigger type code (written into FIFO for Master to identify) |

Generic trigger infrastructure ports (passed through to `trig_algo_support`):

| Signal | Dir | Width | Description |
|---|---|---|---|
| `TRIGGER_ENABLE` | in | 1 | VME enable bit |
| `TRIGGER_VETO` | in | 1 | Veto input (if set, no triggers issued) |
| `TRIGGER_HOLDOFF` | in | 16 | Self-veto duration after firing (×20 ns ticks) |
| `TRIG_HOLDOFF_ENBL` | in | 1 | Enable holdoff logic |
| `TRIGGER_PRESCALE` | in | 16 | Skip this many triggers after each accepted one |
| `TIME_STAMP_BUS` | in | 48 | Running timestamp (captured at trigger time) |
| `TRIG_DIST_MASK` | in | 8 | Which digitizers/Routers receive this trigger |
| `TRIG_FIFO_RE` | in | 1 | Read enable from Master collector state machine |
| `TRIG_FIFO_ACK` | in | 1 | Collector acknowledge (event consumed) |
| `TRIG_FIFO_OUT` | out | 16 | Trigger FIFO data to Master collector |
| `EVENT_AVAILABLE` | out | 1 | Triggers pending in FIFO |
| `RAW_NONVETOED_TRIG_ACK` | out | 1 | Pulse: threshold crossed, veto honored, enable ignored (for matrix use) |
| `ENABLED_TRIG_ACK` | out | 1 | Pulse: threshold crossed + enabled (veto ignored; for rate counting) |
| `ENABLED_NONVETOED_TRIG_ACK` | out | 1 | Pulse: threshold crossed + enabled + not vetoed (actual trigger accept) |
| `ALGO_THROTTLE_REQUEST` | out | 1 | FIFO >50% full; request throttle |
| `MON_EVENT_AVAILABLE` | out | 1 | Trigger monitor FIFO has data |
| `TRIG_MON_FIFO_DATA_OUT` | out | 16 | Trigger monitoring shadow FIFO read port |

## Key Logic / State Machine

### TOTALPROC — 2-state machine
**WAIT_TRIG** (armed, waiting):  
- If `SUM_OF_X > SUM_OF_X_THRESH`: set `TRIGGER_OCCURRED='1'`, go to WAIT_FALL  
- Else: `TRIGGER_OCCURRED='0'`, stay  

**WAIT_FALL** (counting one trigger, waiting to re-arm):  
- `TRIGGER_OCCURRED='0'` (one-tick pulse has been issued)  
- If `SUM_OF_X > SUM_OF_X_THRESH`: stay in WAIT_FALL  
- Else: return to WAIT_TRIG (re-arm)  

This produces exactly one `TRIGGER_OCCURRED` pulse per threshold-crossing event, regardless of how long the sum stays above threshold.

### trig_algo_support (U2)
A generic sub-component shared by all MTRG trigger algorithms. Receives `TRIGGER_OCCURRED` and handles:
- Enabling/disabling based on `TRIGGER_ENABLE`
- Veto check via `TRIGGER_VETO`
- Self-holdoff: after firing, suppresses new triggers for `TRIGGER_HOLDOFF × 20 ns` if `TRIG_HOLDOFF_ENBL='1'`
- Prescaling: skips `TRIGGER_PRESCALE` triggers between accepted ones
- Timestamp capture into trigger FIFO
- FIFO fullness monitoring → `ALGO_THROTTLE_REQUEST`
- Three acknowledge signals (`RAW_NONVETOED`, `ENABLED`, `ENABLED_NONVETOED`) for different counting purposes
- Trigger monitoring shadow FIFO (separate read port for diagnostic readout)

## Key Constants / Parameters
- Comparison: `>` (strictly greater than), not `>=`
- Threshold width: 16-bit (SUM_OF_X_THRESH), accommodates sums from many Routers
- Holdoff added 2025-10-22; prescale was present earlier

## Connections to Other Modules
- **Receives from**: `calc_total_sum.vhd` (SUM_OF_X = total X-plane sum across all Routers); VME registers (threshold, enable, prescale, holdoff, type code)
- **Sends to**: Master Trigger collector state machine (TRIG_FIFO_OUT, EVENT_AVAILABLE); throttle logic (ALGO_THROTTLE_REQUEST); matrix trigger logic (RAW_NONVETOED_TRIG_ACK)
- **Instantiates**: `trig_algo_support` (generic trigger support, not detailed here)
- **Note**: An equivalent Y-plane version likely exists; this file handles the X-plane sum trigger
